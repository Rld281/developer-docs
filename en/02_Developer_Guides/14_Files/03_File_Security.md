---
title: File security
summary: Manage access permission to assets
icon: lock
---

# File security

## Overview

File security is an important concept, and is as essential as managing any other piece of data that exists
in your system. As pages and dataobjects can be either versioned, or restricted to view by authenticated
members, it is necessary at times to apply similar logic to any files which are attached to these objects
in the same way.

## Definitions

There's two dimensions in which to classify how a file can be accessed.

Versioning stage:

- "Draft file" (default): A file which hasn't been published (default after upload).
   A subset of "protected file". See [versioning](/developer_guides/model/versioning).
- "Published file": A published file (can be protected by further access restrictions).
   Files are often published indirectly as part
   of the objects who own them (see [File Ownership](file_management#ownership)).

Access restrictions:

- "Unprotected file" (default): A file without access restrictions.
- "Protected file": A file with access restrictions.
   Note that draft files are always protected, and even published files
   can be protected if they have access restrictions.

## Permission model

Like all other objects in Silverstripe CMS, permissions are generally controlled via `can*()` methods,
for example `canView()` (see [permissions](/developer_guides/security/permissions)).

The permission model defines the following actions:

- View: Access file metadata in the database.
- Edit: Edit file metadata as well as replacing the file content.
- Create: Create file metadata and upload file content.
- Delete: Delete file metadata and the file content.
- Download: Access the file content, but not the file metadata.
   Usually treated the same as "View".

There's a few rules guiding their access, in descending order of priority:

- Published and unprotected files can be downloaded by anyone knowing the URL.
   They bypass any Silverstripe CMS permission checks (served directly by the webserver).
- Access can be restricted by custom `can*()` method implementations on `File`
   (through [extensions](/developer_guides/extending/extensions)).
   This logic can overrule any further restrictions below.
- Users with "Full administrative rights" (`ADMIN` permission code)
   have view and edit access by default, regardless of further restrictions below.
- Users with "Edit any file" permissions (`FILE_EDIT_ALL` permission code)
   have edit access by default, regardless of further restrictions below.
- View or edit access can be restricted per file or folder through
   an inherited permissions model similar to page content (through [`InheritedPermissionsExtension`](api:SilverStripe\Security\InheritedPermissionsExtension)).
   There are five types: "Inherit from parent" (default), "Anyone", "Logged-in users", "Only these groups", or "Only these users".
- Protected files (incl. draft files) allow view/edit access when `File::$non_live_permissions` is satisfied.
   By default, that's configured for anyone with access to any CMS section, or
   the ability to "view draft content".

- Protected files need an "access grant" for the current session
   in order to download the file (see [User access control](#user-access-control)).
   While you can technically allow viewing or editing a file without granting
   access to download it, those aspects are usually bundled together by the file viewing logic.

Access to create or delete files generally aligns with the edit access described above.

Note that even if the permissions above allow access,
you need to have access to a mechanism to view or edit file information.
Most commonly this is through the "Access to Files section" permission.
Custom implementations (e.g. APIs or custom file viewers) can have
further restrictions in your project.

[warning]
When implementing your own `canView()` logic through [extensions](/developer_guides/extending/extensions),
existing unprotected files are not retroactively moved to the protected asset store.
While those new permissions are honoured in the CMS, protected files through custom `canView()`
can still be downloaded through a public URL until a `write()` operation is triggered on them.
[/warning]

## Asset stores

Out of the box, Silverstripe CMS comes with two asset stores: a public and a protected one.
Most operations which act on assets work independently of this mechanism,
without having to consider whether any specific file is protected or public, but can normally be
instructed to favour private or protected stores in some cases.

For instance, in order to write an asset to a protected location you can use the following additional
config option:

```php
$store = singleton(AssetStore::class);
$store->setFromString('My protected content', 'my-folder/my-file.jpg', null, null, [
    'visibility' => AssetStore::VISIBILITY_PROTECTED,
]);
```

## User access control {#user-access-control}

Access for files is granted on a per-session basis, rather than on a per-member basis, via
whitelisting accessed assets. This means that access to any protected asset must be made prior to the user
actually attempting to download that asset. This is normally done in the PHP request that generated
the response containing the link to that file.

An automated system will, in most cases, handle this whitelisting for you. Calls to getURL()
will automatically whitelist access to that file for the current user. Using this as a guide, you can easily
control access to embedded assets at a template level.

```ss
<ul class="files">
    <% loop $File %>
        <% if $canView %>
            <li><a href="$URL">Download $Title</a></li>
        <% else %>
            <li>Permission denied for $Title</li>
        <% end_if %>
    <% end_loop >
</ul>
```

Users who are able to guess the value of $URL will not be able to access those urls without being
authorised by this code.

In order to ensure protected assets are not leaked publicly, but are properly whitelisted for
authorised users, the following should be considered:

Caching mechanisms which prevent `$URL` being invoked for the user's request (such as `$URL` within a
partial cache block) will not whitelist those files automatically. You can manually whitelist a
file via PHP for the current user instead, by using the following code to grant access.

```php
namespace {
    use SilverStripe\CMS\Controllers\ContentController;

    class PageController extends ContentController
    {
        public function init()
        {
            parent::init();

            // Whitelist the protected files on this page for the current user
            $file = $this->File();
            if ($file->canView()) {
                $file->grantFile();
            }
        }
    }
}
```

If a user does not have access to a file, you can still generate the URL but suppress the default
permission whitelist by invoking the getter as a method, but pass in a falsey value as a parameter.
(or '0' in template as a workaround for all parameters being cast as string)

```ss
<% if not $canView %>
    <%-- The user will be denied if they follow this url --%>
    <li><a href="$getURL(0)">Access to $Title is denied</a></li>
<% else %>
    ...
<% end_if %>
```

Alternatively, if a user has already been granted access, you can explicitly revoke their access using
the `revokeFile` method.

```php
namespace {
    use SilverStripe\CMS\Controllers\ContentController;

    class PageController extends ContentController
    {
        public function init()
        {
            parent::init();

            // Whitelist the protected files on this page for the current user
            $file = $this->File();
            if ($file->canView()) {
                $file->grantFile();
            } else {
                // Will revoke any historical grants
                $file->revokeFile();
            }
        }
    }
}
```

## Controlling asset visibility

The asset API provides three main mechanisms for setting the visibility of an asset. Note that
these operations are applied on a per file basis, and unlike `revoke` or `grant` methods
these do not affect visibility for specific users.

Visibility can be specified when files are created via one of the `AssetStore::VISIBILITY_PROTECTED`
or `AssetStore::VISIBILITY_PUBLIC` constants. It's advisable to ensure the visibility of any file
is declared as early as possible, so that potentially sensitive content never touches any
public facing area.

For example:

```php
$object->MyFile->setFromLocalFile($tmpFile['Path'], $filename, null, null, [
    'visibility' => AssetStore::VISIBILITY_PROTECTED,
]);
```

You can also adjust the visibility of any existing file to either public or protected.

```php
// This will make the file available only when a user calls `->grant()`
$object->SecretFile->protectFile();

// This file will be available to everyone with the URL
$object->PublicFile->publishFile();
```

[notice]
One thing to note is that all variants of a single file will be treated as
a single entity for access control, so specific variants cannot be individually controlled.
[/notice]

## How file access is protected

Filesystem paths can change depending if the file is protected or public,
but its public URL stays the same. You just need to use Silverstripe CMS's APIs to generate URLs to those files.
Similarly, operations which modify files do not normally need to be told whether the file is protected or public
either. This provides a consistent method for interacting with files.

In day to day operation, moving assets to or between either of these stores does not normally
alter these asset urls, as the routing mechanism will infer file access requirements dynamically.
This allows you to prepare predictable file urls on a draft site, which will not change once
the page is published, but will be protected in the mean time.

For instance, consider two files `OldCompanyLogo.gif` in the public store, and `NewCompanyLogo.gif`
in the protected store, awaiting publishing.

Internally your folder structure would look something like:

```text
assets/
    OldCompanyLogo.gif
    .protected/
        .htaccess
        a870de278b/
            NewCompanyLogo.gif
```

The urls for these two files, however, do not reflect the physical structure directly.

- The public file at `https://www.example.com/assets/OldCompanyLogo.gif` will be served directly from the web server,
  and will not invoke a PHP request.
- The protected file at `https://www.example.com/assets/a870de278b/NewCompanyLogo.gif` will be routed via a 404 handler to PHP,
  which will be passed to the [`ProtectedFileController`](api:SilverStripe\Assets\Storage\ProtectedFileController) controller, which will serve
  up the content of the hidden file, conditional on a permission check.

When the file `NewCompanyLogo.gif` is made public, the file
will be moved to `assets/NewCompanyLogo.gif`, and will be served directly via
the web server, bypassing the need for additional PHP requests.

```php
use SilverStripe\Assets\Storage\AssetStore;

$store = singleton(AssetStore::class);
$store->publish('NewCompanyLogo.gif', 'a870de278b475cb75f5d9f451439b2d378e13af1');
```

After this the filesystem will now look like below:

```text
assets/
    NewCompanyLogo.gif
    .protected/
        .htaccess
        33be1b95cb/
            OldCompanyLogo.gif
```

## Performance considerations

In order to ensure that a site does not invoke any unnecessary PHP processes when serving up files,
it's important to ensure that your server is configured correctly. Serving public files via PHP
will add unnecessary load to your server, but conversely, attempting to serve protected files
directly may lead to necessary security checks being omitted.

See the web server setting section below for more information on configuring your server properly

### Performance: static caching

If you are deploying your site to a server configuration that makes use of static caching, it's essential
that you ensure any page or dataobject cached adequately publishes any linked assets. This is due to the
fact that static caching will bypass any PHP request, which would otherwise be necessary to whitelist
protected files for these users.

This is especially important when dealing with draft content, as frontend caches should not attempt to
cache protected content being served to authenticated users. This can be achieved by configuring your cache
correctly to skip `Pragma: no-cache` headers and the `bypassStaticCache` cookie.

## Configuring protected assets

In most cases, developers can interact with File and Image objects without worrying about how
Silverstripe CMS resolves file names or responds to requests. Some advanced use cases may occasionally
require developers to adjust the HTTP response for file requests.

Most of the routing logic for serving Files is controlled via the `AssetStore` interface. The default
implementation of the `AssetStore` is `FlysystemAssetStore`.

### Configuring: protected folder location

In the default Silverstripe CMS configuration, protected assets are placed within the web root into the
`assets/.protected` folder, into which is also generated a `.htaccess` or `web.config` configured
to deny any and all direct web requests.

In order to better ensure these files are protected, it's recommended to move this outside of the web
root altogether.

For instance, given your web root is in the folder `/sites/myapp/www`, you can tell the asset store
to put protected files into `/sites/myapp/protected` with the below `.env` setting:

```bash
SS_PROTECTED_ASSETS_PATH="/sites/myapp/protected"
```

### Configuring: protected file headers {#protected_file_headers}

In certain situations, it's necessary to customise HTTP headers required either by
intermediary caching services, or by the client, or upstream caches.

When a protected file is served it will also be transmitted with all headers defined by the
`SilverStripe\Filesystem\Flysystem\FlysystemAssetStore.file_response_headers` config.
You can customise this with the below config:

```yml
SilverStripe\Filesystem\Flysystem\FlysystemAssetStore:
  file_response_headers:
    Pragma: 'no-cache'
```

### Configuring file HTTP response code

When a user tries to access a file that exists, but for which they do not have access,
Silverstripe CMS will return a "404 Not found" response rather than a "403 Denied" to
avoid revealing the existence of the file.

You can customise the response codes for various types of requests via
configuration flags on `FlysystemAssetStore`.

```yml
SilverStripe\Filesystem\Flysystem\FlysystemAssetStore:
  denied_response_code: 403 # The default for this is 404
  missing_response_code: 404
  redirect_response_code: 302
  permanent_redirect_response_code: 301
```

### Updating a file HTTP response before it's sent back to the browser

`silverstripe/assets` 1.6 and above allows you to intercept the file HTTP response
before it's sent to the client by applying an `Extension` to `FlysystemAssetStore`.

To achieve this create an `Extension` and implement the `updateResponse` method.

```php
namespace App\Extension;

use SilverStripe\Control\HTTPResponse;
use SilverStripe\Core\Extension;

class AssetStoreExtension extends Extension
{
    /**
     * @param HTTPResponse $response Update this object to modify the response
     * @param string $asset Path of the request minus the `assets` prefix
     * @param array $context This array contains some resolution information from
     *   FlysystemAssetStore. It may be empty. It may contain a `visibility` key
     *   to say if we are serving a public or protected file. It may contain a
     *   `parsedFileID` detailing how FlysystemAssetStore has resolved $asset.
     */
    public function updateResponse(
        HTTPResponse $response,
        string $asset,
        array $context
    ): void {
        // Do something to the response
    }
}
```

Enable the extension with YAML configuration:

```yml
SilverStripe\Assets\Flysystem\FlysystemAssetStore:
  extensions:
    - App\Extension\AssetStoreExtension
```

### Configuring: archive behaviour

By default, the default extension `AssetControlExtension` will control the disposal of assets
attached to objects when those objects are archived or replaced. For example, unpublished versioned objects
will automatically have their attached assets moved to the protected store. The archive of
draft or (or deletion of unversioned objects) will have those assets permanently deleted
(along with all variants).

Note that regardless of this setting, the database record will still be archived in the
version history for all Versioned DataObjects.

In some cases, it may be preferable to have any assets retained for archived versioned dataobjects,
instead of deleting them. This uses more disk storage, but will allow the full recovery of archived
records and files.

This can be applied to DataObjects on a case by case basis by setting the `keep_archived_assets`
config to true on that class. Note that this feature only works with dataobjects with
the `Versioned` extension.

```php
namespace App\Model;

use SilverStripe\ORM\DataObject;
use SilverStripe\Versioned\Versioned;

class MyVersionedObject extends DataObject
{
    /** Ensure assets are archived along with the DataObject */
    private static $keep_archived_assets = true;

    private static $extensions = [
        Versioned::class,
    ];
}
```

The extension can also be globally disabled by removing it at the root level:

```yml
SilverStripe\ORM\DataObject:
  AssetControl: null
```

## Webserver configuration

### Protected files location

Protected files are stored in `public/assets/.protected` by default
(assuming you're using the [public/ subfolder](/getting_started/directory_structure)).
While default configuration is in place to avoid the webserver serving these files,
we recommend moving them out of the webroot altogether -
see [Server Requirements: Secure Assets](/getting_started/server_requirements#secure-assets).

### Config templates

If the default server configuration is not appropriate for your specific environment, then you can
further customise the `.htaccess` or `web.config` by editing one or more of the below:

- `PublicAssetAdapter_HTAccess.ss`: Template for public permissions on the Apache server.
- `PublicAssetAdapter_WebConfig.ss`: Template for public permissions on the IIS server.
- `ProtectedAssetAdapter_HTAccess.ss`: Template for the protected store on the Apache server (should deny all requests).
- `ProtectedAssetAdapter_WebConfig.ss`: Template for the protected store on the IIS server (should deny all requests).

Each of these files will be regenerated on `?flush`, so it is important to ensure that these files are
overridden at the template level, not via manually generated configuration files.

#### Apache

In order to ensure that public files are served correctly, you should check that your `assets/.htaccess`
bypasses PHP requests for files that do exist. The default template
(declared by `PublicAssetAdapter_HTAccess.ss`) has the following section, which may be customised in your project:

```text
# Non existent files passed to requesthandler
RewriteCond %{REQUEST_URI} ^(.*)$
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule .* ../index.php [QSA]
```

You will need to ensure that your core apache configuration has the necessary `AllowOverride`
settings to support the local `.htaccess` file.

Although assets have a 404 handler which routes to a PHP handler, `.php` files within assets itself
should not be allowed to be marked as executable.

When securing your server you should ensure that you protect against both files that can be uploaded as
executable on the server, as well as protect against accidental upload of `.htaccess` which bypasses
this file security.

For instance your server configuration should look similar to the below:

```xml
<Directory "/var/www/superarcade/public/assets">
  php_admin_flag engine off
</Directory>
```

The `php_admin_flag` will protect against uploaded `.htaccess` files accidentally re-enabling script
execution within the assets directory.

#### Windows IIS 7.5+

Configuring via IIS requires the Rewrite extension to be installed and configured properly.
Any rules declared for the assets folder should be able to dynamically serve up existing files,
while ensuring non-existent files are processed via the Framework.

The default rule for IIS is as below (only partial configuration displayed):

```xml
<rule name="Secure and 404 File rewrite" stopProcessing="true">
    <match url="^(.*)$" />
    <conditions>
        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
        <add input="../index.php" matchType="IsFile" />
    </conditions>
    <action type="Rewrite" url="../index.php" appendQueryString="true" />
</rule>
```

You will need to make sure that the `allowOverride` property of your root web.config is not set
to false, to allow these to take effect.

#### Other server types

If using a server configuration which must be configured outside of the web or asset root, you
will need to make sure you manually configure these rules.

For instance, this will allow your nginx site to serve files directly, while ensuring
dynamic requests are processed via the Framework:

```text
location ^~ /assets/ {
    sendfile on;
    try_files $uri index.php?$query_string;
}
```
