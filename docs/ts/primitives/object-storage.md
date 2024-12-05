---
seotitle: Using Object Storage in your backend application
seodesc: Learn how you can use Object Storage to store files and unstructured data in your backend application.
title: Object Storage
subtitle: Simple and scalable storage APIs for files and unstructured data
infobox: {
  title: "Object Storage",
  import: "encore.dev/storage/objects",
}
lang: ts
---

Object Storage is a simple and scalable solution to store files and unstructured data in your backend application.

The most common implementation is Amazon S3 ("Simple Storage Service") and its semantics are universally supported by every major cloud provider.

Encore.ts provides a cloud-agnostic API for working with Object Storage, allowing you to store and retrieve files with ease. It has support for Amazon S3, Google Cloud Storage, as well as any other S3-compatible implementation (such as DigitalOcean Spaces, MinIO, etc.).


Additionally, when you use Encore's Object Storage API you also automatically get:

* Automatic tracing and instrumentation of all Object Storage operations
* Built-in local development support, storing objects on the local filesystem
* Support for integration testing, using a local, in-memory storage backend

## Creating a Bucket

The core of Object Storage is the **Bucket**, which represents a collection of files.
In Encore, buckets must be declared as package level variables, and cannot be created inside functions.
Regardless of where you create a bucket, it can be accessed from any service by referencing the variable it's assigned to.

When creating a bucket you can configure additional properties, like whether the objects in the bucket should be versioned.

For example, to create a bucket for storing profile pictures:

```ts
import { Bucket } from "encore.dev/storage/objects";

export const profilePictures = new Bucket("profile-pictures", {
  versioned: false
});
```

## Uploading files

To upload a file to a bucket, use the `upload` method on the bucket variable.

```ts
const data = Buffer.from(...); // image data
const attributes = await profilePictures.upload("my-image.jpeg", data, {
  contentType: "image/jpeg",
});
```

The `upload` method additionally takes an optional `UploadOptions` parameter
for configuring additinal options, like setting the content type (see above),
or to reject the upload if the object already exists.


## Downloading files

To download a file from a bucket, use the `download` method on the bucket variable:

```ts
const data = await profilePictures.download("my-image.jpeg");
```

The `download` method additionally takes a set of options to configure the download,
like downloading a specific version if the bucket is versioned.

## Listing objects

To list objects in a bucket, use the `list` method on the bucket variable.

It returns an async iterator of `ListEntry` objects that you can use to easily
iterate over the objects in the bucket using a `for await` loop.

For example, to list all profile pictures:

```ts
for await (const entry of profilePictures.list({})) {
  // Do something with entry
}
```

The `ListOptions` type can be used to limit the number of objects returned,
or to filter them to a specific key prefix.

## Deleting objects

To delete an object from a bucket, use the `remove` method on the bucket variable.

For example, to delete a profile picture:

```ts
await profilePictures.remove("my-image.jpeg");
```

## Retrieving object attributes

You can retrieve information about an object using the `attrs` method on the bucket variable.
It returns the attributes of the object, like its size, content type, and ETag.

For example, to get the attributes of a profile picture:

```ts
const attrs = await profilePictures.attrs("my-image.jpeg");
```

For convenience there is also `exists` which returns a boolean indicating whether the object exists.

```ts
const exists = await profilePictures.exists("my-image.jpeg");
```

## Error handling

The methods throw exceptions if something goes wrong, like if the object doesn't exist or the operation fails.

If an object does not exist, it throws an `ObjectNotFound` error.

If an upload fails due to a precondition not being met (like if the object already exists
and the `notExists: true` option is set), it throws a `PreconditionFailed` error.

Other errors are returned as `ObjectsError` errors (which the above errors also extend).

## Bucket references

Encore uses static analysis to determine which services are accessing each bucket,
and what operations each service is performing.

That information is used for features such as rendering architecture diagrams, and is used by Encore Cloud to provision infrastructure correctly and configure IAM permissions.

This means `Bucket` objects can't be passed around however you like,
as it makes static analysis impossible in many cases. To simplify your workflow, given these restrictions,
Encore supports defining a "reference" to a bucket that can be passed around any way you want.

### Using bucket references

Define a bucket reference by calling `bucket.ref<DesiredPermissions>()` from within a service, where `DesiredPermissions` is one of the pre-defined permission types defined in the `encore.dev/storage/objects` module.

This means you're effectively pre-declaring the permissions you need, and only the methods that
are allowed by those permissions are available on the returned reference object.

For example, to get a reference to a bucket that can only download objects:

```typescript
import { Uploader } from "encore.dev/storage/objects";
const ref = profilePictures.ref<Uploader>();

// You can now freely pass around `ref`, and you can use
// `ref.upload()` just like you would `profilePictures.upload()`.
```

To ensure Encore still is aware of which permissions each service needs, the call to `bucket.ref`
must be made from within a service, so that Encore knows which service to associate the permissions with.

Encore provides permission interfaces for each operation that can be performed on a bucket:

* `Downloader` for downloading objects
* `Uploader` for uploading objects
* `Lister` for listing objects
* `Attrser` for getting object attributes
* `Remover` for removing objects

If you need multiple permissions you can combine them using `&`.
For example, `profilePictures.ref<Downloader & Uploader>` gives you a reference
that allows calling both `download` and `upload`.

For convenience Encore also provides a `ReadWriter` permission that gives complete read-write access
to the bucket, granting all the permissions above. It is equivalent to `Downloader & Uploader & Lister & Attrser & Remover`.