---
authors: Jordan Hendricks <jordan.hendricks@joyent.com>
state: draft
---

<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright 2016 Joyent Inc.
-->

# RFD 65 Multipart Uploads for Manta

## Introduction

This document will describe the proposed design and implementation of a multipart upload mechanism for Manta, similar to Amazon s3's multipart upload API.

In section 1, we discuss the proposed design for this feature in detail: first outlining desired operations for a multipart upload API, then reviewing design constraints and requirements for implementing such a feature in Manta, and finally, reasoning through each piece of the design. If you are interested in the motivation behind different design designs for Manta multipart uploads, you should read this section.

In section 2, we provide a more concise summary of the design: the precise API calls and how they map to an HTTP request, as well as a summary of the required Manta metadata to achieve this design. If you only want to see a summary of the API, skip to this section.

Finally, in section 3, we revisit the desired operations and design constraints and requirements proposed in section 1 and summarize how these are addressed in the final design.


## 1. Design Discussion

### Multipart Upload Description

A multipart upload API allows a user of an object store to break up an object into distinct parts and upload them over time to create a single object in the object store. Such a feature may be useful for devices uploading large objects over volatile network connections or for allowing for parallelism uploading a large object with multiple devices.

### Desired Multipart Upload Operations

At a minimum, there are a few things a consumer of a multipart upload API will need to do:
* Initiate a multipart upload for an object
* Upload the parts of that object
* Indicate that the upload is completed after all parts are uploaded

In addition to these operations, some others that are useful are:
* Canceling an upload in progress
* Querying an upload's state
* Listing parts that have been uploaded for an upload 
* See all multipart uploads in progress

These operations are all available in Amazon's s3 storage API for multipart uploads.


### Implementation Requirements and Constraints

From a Manta implementation perspective, at a minimum, we need the following things to implement a multipart upload API:

1. A place to store the parts as they are uploaded
2. A mechanism to create the final object from the uploaded parts

Additionally, to make the full suite of desired multipart upload operations work well, we need to:

3. Ensure that parts can be listed easily
4. Ensure that uploads can be listed easily
5. Account for multiple clients using the API and various component failures (especially for the case of completing an upload)
6. Account for cleanup of upload metadata and part data

Finally, we want to reuse existing Manta code wherever possible.

In terms of API constraints, we will adopt many of Amazon s3's general constraints for multipart uploads:
* Parts may be uploaded in any order
* There must be a consecutive set of parts from `{0..N}` specified on the commit of the object
* An upload has a maximum of 10,000 parts
* Parts have a minimum size of 5mb


### Implementation Discussion

With these goals and constraints in mind, we can now begin to design the multipart upload implementation in Manta.

#### Storing Parts
Similar to the Manta jobs API, which stores information about jobs at `/$ACCOUNT/jobs` in Manta, we can store multipart uploads under a special uploads directory at the root of the account. 

One way to do this is to create a single directory for each upload, with the the upload ID as the directory name, and store the parts for the upload in its directory. Having a well-defined directory structure with uploads in single directory and the parts of an upload also in a single directory is desirable, as it allows both uploads and their parts to be listed easily. But keeping all upload directories in the same parent directory also has a drawback: the upper bound of ongoing multipart uploads for a given account is fixed at the number of allowed directory entries (a few million).

To circumvent this limitation while still limiting the number of shards upload metadata records live on, we can compromise a bit. We distribute the uploads into a fixed number of subdirectories within the special `uploads` directory, with each subdirectory as one possible character of a uuid. An upload is stored in the subdirectory that is the first character of its uuid. 

More concretely, the path to a given upload directory is thus: `/$ACCOUNT/uploads/[0-f]/<upload uuid>`. 

Parts are then stored as: `/$ACCOUNT/uploads/[0-f]/<upload uuid>/partN`, where `N` is the part number of the part.

This approach ensures that a maximum of 16 shards need to be contacted to list all ongoing uploads for an account. It also gives us flexibility to distribute uploads across more directories in the future if we find that this doesn't provide enough capacity for ongoing uploads for an account -- we can extend the subdirectory structure to include more uuid characters (e.g., the first two chars).

#### Creating the Final Object: `mako-finalize`

The completed object is a concatenation of all of the upload's parts, which are finalized on commit. Because objects in Manta are immutable, this operation will require copying data at some point — we cannot simply append parts as they are uploaded to an existing object. Additionally, a multipart upload object could be very large, so we want to avoid copying parts over the network if possible. 

As such, we will store all of the parts on all the storage nodes that the final object will live on. This way, the object can be constructed on the storage node itself, and copying data will only occur locally. This approach requires selecting the set of storage nodes the object will reside on when the upload is initiated, and ensuring that all parts are uploaded to this set.

When the upload is committed, we must do two things: idempotently create the object on disk on the storage node, as if it was placed there in a normal object PUT in Manta, and remove the parts to avoid taking up unnecessary space on the node. It is important that the construction of these two operations be idempotent, in case the storage node restarts and the operation has to be retried.

To do so, we propose a new operation: `mako-finalize`. In this mechanism, the storage node first concatenates the object parts to be committed to the object — these are specified as a parameter to the commit. Once the storage node has finished writing the object, it removes the parts, as it is now safe to do so. Finally, `mako-finalize` will return the md5sum of the object, so that it can be stored in the metadata record of the committed object.

The idempotency of `mako-finalize` is critical here. If the storage node crashes before or in the middle of writing the object, `mako-finalize` can still reconstruct it, as none of the parts have yet been removed. Importantly, this presume a restriction on committing uploads: after a commit has begun, the set of parts input to the commit cannot change, even if the commit is retried. Otherwise, we could get into a state where an uploaded object with one set of parts is created, the parts removed from the mako node, and the commit operation fails at a later step, leaving the client to retry the commit. At this point, the parts have been cleaned up. 

Additionally, `mako-finalize` will also need to the number of bytes the final object will be as input, to differentiate between a partially written object or a completed one.

The function signature for this operation will look something like the following:

```
mako-finalize(uploadId, account, objectId, nbytes, parts[]) -> md5sum
```
In pseudocode, `mako-finalize` does the following steps:

```
mako-finalize:
	If objectId exists, then:
		If size of objectId is less than nbytes, then:
			Write objectId file from parts files

	Else:
		For part in parts[]:
			Append part to objectId file


	Remove parts files

	Return: md5sum(objectId file)
```

#### Handling Concurrent Requests

For most desired multipart upload operations, multiple clients attempting to perform the same operation do not present any issues:
* initiating an upload should return a unique upload ID for each client, even if the uploads are for the same path
* uploading a part will replace an existing part for a given upload if there is already one uploaded (analagous to a normal Manta PUT)
* listing uploads or parts is a metadata read (no state change occurs)

The more interesting operations here are committing and aborting an upload. It is important that both commit and abort be atomic operations, such that if multiple clients attempt either on the same upload ID, only one client will win (and the other receives a meaningful response). Relatedly, we also need to account for failures such that commits and aborts are idempotent operations — if the response to a commit or abort is lost, the client needs some way to observe that the commit or abort has succeeded. One way to do so is to make the operations idempotent.

In the case of a commit, there are several steps that need to happen:
* Insert an object record for the uploaded object.
* Create the object from its parts on the set of storage nodes (`mako-finalize`).
* Update some metadata to indicate the operation has been committed, such that if another client tries to commit or abort, we can handle the request correctly.

In order for a commit to be atomic, the state of the upload must be able to be queried on the same shard as the object record — it is not sufficient to store the state on the shard of the upload directory record, which could be a different shard, nor can the state be contained in the object record itself, as the object could be deleted immediately after the upload is finished.

A clear solution is to atomically write a commit or abort record on the same shard the object record is inserted on. If a commit or abort record already exists for the same object ID, then the insertion of these record(s) will fail. This approach ensures only one client will win the commit/abort race.

This approach also has a minor setback: when querying the status of a given upload ID, we must always contact two different shards — the shard of `$ACCOUNT/uploads/[0-f]`, to see if the upload exists, and the shard mapping to the object path, to check if a commit or abort record exists for the upload ID. Given that most of the lifetime of an upload will not be in a committed/aborted state, it would be nice if we didn't have to do this extra lookup for every query. 

Instead, we will also add metadata to the upload record that records some state. As discussed, we cannot keep track of the state of the upload once a commit or abort has tarted (because the final state of the upload must be atomically recorded on the shard of the final object record), but we can at least differentiate between the upload being in progress, by virtue of it having been created, and a commit/abort operation having been started on the upload. Once a commit/abort has begun, the current state of the upload _must_ be queried on the shard of the object record.

More succinctly, we can describe the states of the upload record as: `CREATED` and `FINALIZING`. If the state is `CREATED`, then the upload has been started and is in progress. In this case, there is no need to query any other shards when checking the state of a given upload ID. Once a client calls `commit` or `abort`, then the first step of the commit/abort operations is to update the state in the upload record to `FINALIZING`, before doing any other operations. A state of `FINALIZING` indicates that the true status of the multipart upload now lives on a different shard than that of the upoad record. 

Additionally, as detailed in `mako-finalize`, we need to enforce that the set of parts specified on the first commit on an upload ID is the only set of parts that can be committed. This information will also be stored in the upload record when the state is updated to `FINALZING`.

#### Common Case: A normal commit (only one client attempting)

Thus, these are the final steps for a typical commit:

1. Set state metadata of the upload directory record for `$ACCOUNT/uploads/[0-f]/<upload uuid>` to `FINALIZING`, with its type set to `commit`. In this same update, save the part numbers and their etags for the commit.
2. Invoke `mako-finalize` on the storage node set.
3. Atomically insert a commit record and the object record on the shard of `$OBJECT_PATH`, which is the input path for the object upon upload creation.

These are the steps for a typical abort:

1. Set upload record state to `FINALIZING`, and its type to `abort`.
2. Atomically insert an abort record on the shard of `$OBJECT_PATH`.

#### Idempotent Commits and Aborts

Now, we will augment these steps to make commits and aborts idempotent. Because the `mako-finalize` step already is idempotent, we only need to add a check for the upload record state, and if necessary, an existing commit or abort record.

This leaves us with the final design of the `commit` operation:

1. Read the state metadata of the upload directory record for `$ACCOUNT/uploads/[0-f]/<upload uuid>`.
   * If the state is `CREATED`, update this record to have state `FINALIZING`, type `commit`, and record the part numbers and their etags.
   * If the state is `FINALIZING`, type `commit`, then check the part numbers and etags specified in the commit against the metadata. If they don't match, return an error.
2. Invoke `mako-finalize` on the storage node set.
3. If a commit record doesn't exist for the object, then atomically insert a commit record and the object record on the shard of `$OBJECT_PATH`.

And the final design for the `abort` operation:

1. Read the state metadata of the upload directory record for `$ACCOUNT/uploads/[0-f]/<upload uuid>`.
   * If the state is `CREATED`, set it to `FINALIZING`, type `abort`. 
   * If the state is `FINALIZING`, type `commit`, then return an error.
   * If the state is `FINALIZING`, type `abort`, then continue.
2. If an abort record doesn't exist for the object on the shard of `OBJECT_PATH`, then atomically insert an abort record.

Commits and aborts look very similar, except that commits also invoke an operation on the storage. Under this design, once a commit or abort has started, the finalization type of the upload cannot change. Put differently: a commit that has been started cannot be changed to an abort (for instance, after an internal error during the request), but it can be retried. This is true for the flip case: an abort cannot be changed to a commit, but it can be retried. The ability to abort a commit that has returned a 500 may be desirable behavior, but it should be straightforward to add if we decide later that it is desirable.

#### State Machine for Finalizing Uploads
To summarize the states of the multipart upload commit and abort operations, we have abstracted the metadata state recorded in the upload record into the following state machine for simplicity.  Note that `done` is an implicit state (not recorded in the upload record), but guaranteed by the presence of an abort or commit record on the shard of `$OBJECT_PATH`.


```
  mpu-create
      +
      |
      |
      v         mpu-commit                                      write commit/
+-----+-----+   mpu-abort    +------------------------------+   abort record    +------------------------------+
|  created  +--------------> |           finalizing         +-----------------> |             done             |
+-----------+                |   type={aborting,committing} |                   | (committed, aborted, failed) |
                             +------------------------------+                   +------------------------------+

```

### Garbage Collection

We need a mechanism to ensure all unnecessary data associated with multipart uploads is properly removed when they are no longer needed. The items to be cleaned up include:
* Upload directory records
* Part object records
* Commit/abort records
* Part data

All of this data can be removed some period of time after a commit or abort has completed — the status of a commit/abort should still be able to be queried shortly after the commit/abort completes. To do so, we can modify the existing Manta garbage collection process to also clean up these items.

On commits, part data is already cleaned up, but upload records, commit records and part records need to be removed. Aborts need to have all four items removed.

The garbage collection process will thus be modified to do the following steps, represented in psuedocode below.

```
mpu-gc:
	For each upload in /$ACCOUNT/uploads:
		If upload record "state" field is FINALIZING, then:
			Contact the shard of $OBJECT_PATH to see if a commit/abort record exists for the object ID.
			If a commit/abort record exists, then:
				Atomically remove the upload directory record and the parts records for the upload.
				If the record is an abort record, then:
					Invoke the storage node to remove the parts associated with the object ID.
				Remove the commit/abort record.
```

## 2. Final Design

### API Calls

This is a summary of the final API based on the discussion above. Each API calls lists the general inputs and outputs and the high-level steps each operation takes, including some error conditions.

#### 1. `create-mpu`
Creates a new multipart upload.

* Inputs: `$OBJECT_PATH`
* Outputs: upload uuid
* Steps:
  * 1. Generate a new upload uuid, and insert a directory record for `/$ACCOUNT/uploads/[0-f]/uuid` with its state set to `CREATED`.

#### 2. `upload-part`
Uploads data to a given part number of an upload.

* Inputs: upload uuid, part number, part data
* Outputs: etag of part
* Steps:
  * 1. Read the upload state in the object record for `/$ACCOUNT/uploads/[0-f]/uuidd`.
       * If the state is `CREATED`, insert object record for `/$ACCOUNT/uploads/[0-f]/uuid/partN`, where `N` is the part number.
       * If the state is `FINALIZING`, return an error.

Note: This API call has a race between reading the upload record's state and inserting the part record, which could be on a different shard. It is possible that another client starts a commit or abort for the same upload id after the read of the upload record has occurred, so the client uploading a part will not learn that the upload is being finalized (likely with a different set of parts). This case shouldn't be common, as it indicates a strange (and possibly buggy) use of the API. As long as we document this possible case clearly, I think this is acceptable.

#### 3. `commit`
Creates the final object from a set of uploaded parts.

* Inputs: upload uuid, set of part numbers and their associated etags
* Outputs: none
* Steps:
  * 1. Read the state of upload record and check its state.
     * If the state is `CREATED`, read the object record for each input part and verify the etag of the part matches the input etag.
        * If the parts and etags match the input set, then update the upload record state to `FINALIZING`, type `commit`, and save the part numbers and their etags in the upload record.
        * If the parts and etags don't match the input set, then return an error.
     * If the state is `FINALIZING`, type `commit`, then check the part numbers and etags specified in the commit against the metadata. If they don't match, return an error.
     * If the state is `FINALIZING`, type `abort`, then return an error.
  * 2. Invoke `mako-finalize` on the storage node set.
  * 3. If a commit/abort record doesn't exist for the object, then atomically insert a commit record and the object record on the shard of `$OBJECT_PATH`.

#### 4. `abort`
Aborts an upload.

* Inputs: upload uuid
* Outputs: none
* Steps:
  * 1. Read the upload record and check its state.
     * If the state is `CREATED`, set it to `FINALIZING`, type `abort`. 
     * If the state is `FINALIZING`, type `commit`, then return an error.
     * If the state is `FINALIZING`, type `abort`, then continue.
  * 2. If a commit/abort record doesn't exist for the object, then atomically insert an abort record.


#### 5. `list-parts`
Lists the parts uploaded to an upload.

* Inputs: upload uuid
* Outputs: set of part numbers, their associated etags, and other relevant metadata, such as part size
* Steps:
  * 1. Read all records in the directory `/$ACCOUNT/uploads/[0-f]/uuid`.


#### 6. `list-mpu`
Lists all uploads for an account.

* Inputs: none
* Outputs: list of upload uuids (and information about them, such as the `OBJECT_PATH` associated with the upload and the upload state)
* Steps:
  * 1. Read all records in the directory `/$ACCOUNT/uploads`.

#### 7. `get-mpu`
Gets the state for a given upload.

* Inputs: uuid
* Outputs: {`CREATED`,`FINALIZING` type `commit`, `FINALIZING` type `abort`, `DONE`}
* Steps:
  * 1. Read the upload record and return its state.


## HTTP API

The following table demonstrates how to access these API calls over HTTP. The DELETE method is not supported for any operation. An "X" represents an explicitly disallowed action, and a "-" represents an allowed action that doesn't have a corresponding API name.

| | GET | HEAD | PUT | POST |
| --- | :---: | :---: | :---: | :---: |
| __/$ACCOUNT/uploads__ | `list-mpu` | - | X | `create-mpu` |
| __/$ACCOUNT/uploads/[0-f]/uuid__ | `list-parts` | - | X | X |
| __/$ACCOUNT/uploads/[0-f]/uuid/state__ | `get-mpu` | X | X | X |
| __/$ACCOUNT/uploads/[0-f]/uuid/partN__ | X | - | `upload-part` | X
| __/$ACCOUNT/uploads/[0-f]/uuid/commit__ | X | X | X | `commit-mpu` |
| __/$ACCOUNT/uploads/[0-f]/uuid/abort__ | X | X | X | `abort-mpu`

Note that GETs of parts are disallowed. This is important because allowing GETs for part data would allow abuse of the API to store objects on the same storage node, circumventing our mechanism of selecting storage nodes.

## Metadata Summary


This table summarizes the metadata discussed in this design. The "shard" field represents the corresponding Manta shard for the given path.

| | __Upload Record__ | __Part Record__ | __Commit/Abort Record__ | __Object Record__ |
| :---:| --- | --- | --- | --- |
| __Type__ | Directory | Object | Finalizing Record (new type) | Object |
| __Shard__ | `/$ACCOUNT/uploads/[0-f]\<upload uuid\>` | `$/ACCOUNT/uploads/[0-f]/\<upload uuid\>/partN` | `$OBJECT_PATH` | `$OBJECT_PATH` |
| __Bucket__ | MANTA | MANTA | UPLOADS (new bucket) | MANTA |
| __Additional Metadata__ | * State={`CREATED`,`FINALIZING`} <br> * If state=`FINALIZING`, then type={`commit`,`abort`} <br> * If type=`commit`, then part numbers and their etags also included | N/A | N/A | N/A |



## 3. Summary

In the above design discussion, we addressed all of the desired operations for the end user and implementation requirements outlined in section 1. To reiterate, here is an abridged version connecting these initial goals back to the final design:

Desired Operations:
* Initiate a multipart upload for an object: `create-mpu`
* Upload the parts of that object: `upload-part`
* Indicate that the upload is completed after all parts are uploaded: `commit`
* Canceling an upload in progress: `abort`
* Querying an upload's state: `get-mpu`
* Listing parts that have been uploaded for an upload: `list-parts`
* See all multipart uploads in progress: `list-mpu`

Implementation Requirements and Constraints:

1. A place to store the parts as they are uploaded: `/$ACCOUNT/uploads/[0-f]/uuid`
2. A mechanism to create the final object from the uploaded parts: `mako-finalize`
3. Ensure that parts may be listed easily: All parts stored in same directory (and thus their metadata is on the same shard)
4. Ensure that uploads may be listed easily: All uploads listed in a small number of directories (few shards to contact)
5. Account for multiple clients and component failures, particularly when considering finalizing an upload: atomic and idempotent `commit`/`abort`
6. Account for cleanup of upload metadata and part data: garbage collection.

## Affected Repositories
At a minimum, this feature will require changing the following repositories:
* manta-muskie
* manta-mako

I'm not sure which repository this change will affect, but we will also be adding a new bucket to our Postgres database (for the commit and abort records).

## Security Impact
We will be adding new API endpoints through muskie. API calls for multipart upload will be authenticated as other Manta API calls to muskie are. We will also need to consider potential security impact in the implementation of the `mako-finalize` mechanism, since this is an additional interface added to mako.