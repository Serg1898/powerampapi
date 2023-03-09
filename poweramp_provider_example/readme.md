# Poweramp Track Provider API
============================================

Poweramp v3 (build 862+) supports externally provided tracks which are shown along other tracks in the Poweramp Library categories, including Folders and Folders Hierarchy.

This API enables scenarios like providing cloud-based tracks, streamed/cached tracks, file system, http hosted tracks, or other virtual track hierarchies into Poweramp Library.
Poweramp treats such tracks as usual tracks within Poweramp Library categories, they appear the same way the file system tracks appear - Poweramp adds the provider app name label
to such tracks and may treat such tracks slightly differently, as documented here.

Please note that file-backed tracks are most easy to implement in provider: if track file is on the file system somewhere on a device, or,
in other words track has a seekable file descriptor, then features like wave-seekbar, seeking in general, local tag reading are available immediately.
Poweramp can access fill through such seekable file descriptor the same way it accesses its Library tracks.

It's also possible to provide URLs - these can be almost any supported file format on http(s) or hls stream. It's also possible to generate an URL right in the time of the playback,
provide custom headers, cookies, etc.

Your Provider may send track audio data to Poweramp via seekable sockets - similar to sending data via a pipe, but with a seek support via custom protocol.
This approach may be used if your audio data requires special preprocessing, decrypting, or any other processing, when providing
file descriptor to a file on the local storage or http stream directly is not possible.

*Please create github issue if some not-yet-supported streaming format is needed.*

Your provider can provide m3u8/pls playlists and those will be appropriately processed providing http streams in the Poweramp Streams category
and in the Playlists. Simple playlists referencing other tracks, including your provider tracks, are also possible

The Provider API is not completely custom and based on the combination of standard Android APIs:
* [Storage Access Framework/SAF](https://developer.android.com/guide/topics/providers/document-provider)
* [DocumentsContract](https://developer.android.com/reference/android/provider/DocumentsContract)

Due to the standard APIs used, the resulting provider plugin can be used by other apps (such as Google Files, other file managers),
if they implement SAF/DocumentsContract client APIs.

*Please create github issue if you need your provider to be hidden from other apps.*

[TOC levels=3]:# "#Implementing Poweramp Track Provider Plugin"

# Implementing Poweramp Track Provider Plugin
- [Basics](#basics)
- [AndroidManifest.xml](#androidmanifestxml)
- [Scanning](#scanning)
- [EXTRA_LOADING](#extra_loading)
- [Icon](#icon)
- [Provider Tracks In The Poweramp Library](#provider-tracks-in-the-poweramp-library)
- [URL Tracks](#url-tracks)
- [Seekable Sockets, Pipes, File Descriptors, openDocument, CancellationSignal](#seekable-sockets-pipes-file-descriptors-opendocument-cancellationsignal)
- [Playback](#playback)
- [Metadata And Album Art](#metadata-and-album-art)
- [Track Wave](#track-wave)
- [Roots](#roots)
- [Folders Hierarchy And User Selected Sub-folders](#folders-hierarchy-and-user-selected-sub-folders)
- [Sorting](#sorting)
- [Playlists](#playlists)
- [Cue Sheets](#cue-sheets)
- [Data Refresh](#data-refresh)
- [Rescan](#rescan)
- [Deletion](#deletion)
- [Provider Crashes](#provider-crashes)
- [Music Folders Selection Optimization](#music-folders-selection-optimization)


### Basics
Please be sure to review the
[Storage Access Framework/SAF](https://developer.android.com/guide/topics/providers/document-provider)
API basics which define the *Document Provider* (plugin to be implemented), and the *Client app* (Poweramp).

Document Provider provides information about tracks hierarchy and the metainformation (album art, track tags, such as title, album, artist, etc.)
Check this [Document Provider guide](https://developer.android.com/guide/topics/providers/create-document-provider).

Track Provider is added to Poweramp via Settings / Library / *Music Folders* dialog with the plus (+) button. The + button opens standard
Android *Picker* dialog which allows user to select the Tracks Provider Plugin *Root* or a sub-folder.

### AndroidManifest.xml
Your provider AndroidManifest should include `com.maxmpz.PowerampTrackProvider` metadata. See example [AndroidManifest.xml](src/main/AndroidManifest.xml#L23).
Currently (builds 863+) Poweramp can use any document provider available, but this metadata allows Poweramp to understand your Provider supports APIs
described here.


### Scanning

To represent list of provider tracks Poweramp scans Track Provider Plugin in two phases:
* hierarchy scan
  * establishes folder and tracks hierarchy
  * adds new tracks, removes deleted tracks to/from Poweramp Library database
  * updates last modified timestamps
  * projection columns are limited to a few columns
* metadata scan
  * metadata retrieved for the new or modified tracks (based on last modified timestamp)
  * projection columns include most metadata columns, but still exclude some additional possible metadata columns, such as lyrics
* Poweramp also scans one track specifically for Info/Tags or Lyrics UI
  * in this case extra info is expected, such as lyrics or synced lyrics

**Important:** each scan phase is executed in multiple threads. All the related API Provider code should be thread safe
(Android SAF Provider architecture already implies that and requires thread safe implementation).


### EXTRA_LOADING
"Loading" cursors are not supported. Your provider should provide ready to use cursor with the data.
There is little point in requesting scan when appropriate metadata data is not yet loaded by your plugin.
Instead, load data as needed in your plugin and then issue the appropriate Poweramp rescan command.

Alternatively, if data loading is fast enough, just block in your provider method (e.g. queryChildDocuments/queryDocument) and do loading there,
but note that Poweramp scanning has a timeout. There is also a timeout for openDocument call duration which is defined by the user
in Poweramp settings as the "Network Timeout" option.


### Icon

Poweramp shows Provider app icon where appropriate (Music Folders dialog, Info/Tags dialog, etc.)


### Provider Tracks In The Poweramp Library

Poweramp categorizes provider tracks the same way it does that for the usual file system tracks - tracks are available in All Songs, Folders, Folders Hierarchy,
and other categories according to the tags/track metadata, playlists, and playlist stream entries are available in the Playlist and Streams categories.


### URL Tracks

Poweramp supports the "static" and the "dynamic" URL tracks which can be streams or just remote files.

Poweramp looks for non-standard `TrackProviderConsts.COLUMN_URL` column in the provider returned cursor. If the url exists, the track is assumed to be a remote file or a stream.
Whether it's processed as a stream (no duration, non seekable) or a remote file (has duration, seekable), depends on `MediaStore.MediaColumns.DURATION` for the track:
`DURATION` <= 0 defines track as a stream, `DURATION` > 0 makes track a remote track file.

Static URL track has URL defined once when provider is scanned by Poweramp for tracks. URL from `TrackProviderConsts.COLUMN_URL` is used as-is to open the track.
Poweramp doesn't call provider openDocument in this case.

Dynamic URL track forces Poweramp to query actual URL when the track is started in Poweramp. Poweramp does an additional call to the method `TrackProviderConsts.CALL_GET_URL`.
See [ExampleProvider.java](src/main/java/com/maxmpz/powerampproviderexample/ExampleProvider.java#L831) for the example method implementation.

For URL Tracks with the duration, Poweramp will try to pre-scan it for a seek-wave. As this can increase traffic and server load, you can disable this behavior with
`TrackProviderConsts.COLUMN_TRACK_WAVE` set to a zero sized float array: `new float[0]`.

Note, at this moment streamed URL tracks (without duration) are shown by Poweramp in the appropriate Folders Hierarchy in your provider folders. This is different comparing to Poweramp scanned
m3u8 playlists where streams are only visible in Streams, Playlists (incl. smart playlists), and Queue categories.
The Provider stream tracks are also shown in the appropriate Album/Artist/Genre/etc. categories.


### Seekable Sockets, Pipes, File Descriptors, openDocument, CancellationSignal

The SAF API provides access to the data via openDocument method which, in turn, returns file descriptor (wrapped as ParcelFileDescriptor).  
Poweramp accepts the file descriptor pointing to the file somewhere on the local filesystem, or a socket file descriptor.  
A pipe file descriptor is also accepted, but it is not recommended as no pipe seeking is possible.

* the direct file descriptor is the file pointing file descriptor which is seekable with no extra effort. Poweramp is also able
to read tags directly from the track and read embedded album art from it

* the seekable socket descriptor requires a special support
([TrackProviderProtocol.java](../poweramp_api_lib/src/main/java/com/maxmpz/poweramp/player/TrackProviderProto.java)), but
resulting code is very close to the code for a pipe file descriptor. This works on all supported Android versions (5.0+)

* the seekable proxy file descriptor created by StorageManager.openProxyFileDescriptor. Works for Android 8+. Recommended if you're targeting Android 8+

* the pipe file descriptor is also accepted, but is not recommended: no seeking is possible, no tag reading, no embedded album art extraction, file is represented as "stream"

See [ExampleProvider.java openDocument implementation](src/main/java/com/maxmpz/powerampproviderexample/ExampleProvider.java#L572) for the file pointing
file descriptor, the seekable socket, and proxy file descriptor code examples.

Please note that socket is read by Poweramp as much as needed for the track playback and the total reading time from the start to the end of the track is close to the track duration itself.
Poweramp does "blocking" read.
If you download the data you may need to adjust your timeouts or you may download track to a local file cache and/or feed Poweramp with a file pointing file descriptor instead.

CancellationSignal is also provided and can be used to monitor Poweramp closing a file due to the user requesting some other file or due to any other track-ending scenario.
Nevertheless in most cases you'll receive IOException (and Provider should handle it properly) due to the blocking write on your Provider side.


### Playback

Poweramp may open 2 tracks concurrently via your Provider. This is needed to support the crossfade. The same track could be opened concurrently, for example, when
track is the last in a list - Poweramp opens same track again in advance for the next possible playback while finishing playing it.


### Metadata And Album Art

Poweramp supports 2 approaches for the Provider track metadata (tags) and album art:
* metadata and album art image is provided by the Provider
  * this is the only option for URL-based tracks and pipe file descriptors
  * also supported for directories (since 869)
* metadata and album art is retrieved directly from the track file (file descriptor) by Poweramp


#### Metadata And Album Art Provided By Provider

See `DEFAULT_TRACK_AND_METADATA_PROJECTION` in [ExampleProvider.java](src/main/java/com/maxmpz/powerampproviderexample/ExampleProvider.java#L132).

If `MediaStore.MediaColumns.TITLE` or `MediaStore.MediaColumns.DURATION` columns exist in the resulting cursor, Poweramp assumes metadata is provided by the Provider and track is not scanned
for the tags or the album art.

In this case the album art image is retrieved if `Document.COLUMN_FLAGS` column has `Document.FLAG_SUPPORTS_THUMBNAIL` flag.

Poweramp (when needed, e.g. when track is played) requests the album art via `openDocumentThumbnail`. See [ExampleProvider.java](src/main/java/com/maxmpz/powerampproviderexample/ExampleProvider.java#L551)

Lyrics can be also provided via `TrackProviderConsts.COLUMN_TRACK_LYRICS`. Poweramp adds this column to the `projection` columns only when Info/Tags or Lyrics dialog is shown, and
your Provider may take time to extract/download/obtain the lyrics as needed.

(Since 869) Directories may have thumbnail as well, if appropriate `Document.FLAG_SUPPORTS_THUMBNAIL` flag is set for the directory.
See [ExampleProvider.java](src/main/java/com/maxmpz/powerampproviderexample/ExampleProvider.java#L381) and
[ExampleProvider.java](src/main/java/com/maxmpz/powerampproviderexample/ExampleProvider.java#L570)

#### Metadata And Album Art From Track File

If `MediaStore.MediaColumns.TITLE` or/and `MediaStore.MediaColumns.DURATION` are missing from the resulting cursor, Poweramp assumes no metadata is provided and tries to open track file and
scan the tags / retrieve the embedded album art.


### Track Wave

Poweramp uses up to 100 float track amplitude values (-1.0 .. 1.0 range) to show "seek wave" - rough approximation of the track volume over the track duration.
The wave values can be passed by your Provider along other metainformation in the case the metainformation is provided by your provider.

Send 100 float values in range -1.0 .. 1.0 as float[] or byte[] array in `TrackProviderConsts.COLUMN_TRACK_WAVE` column.
If array size is 0, Poweramp assumes default wave (same as used for Streams) is required.
If array size is not 100, Poweramp will resample that to 100 values internally.

You can have either a float[] array (easier to manipulate/generate) or a byte[] array (float array represented as bytes, easier to store/retrieve as BLOB in/from the database).
See [TrackProviderHelper.java](../poweramp_api_lib/src/main/java/com/maxmpz/poweramp/player/TrackProviderHelper.java#L15) for `bytesToFloats`, `floatsToBytes` methods if you need to convert
from one format to the another.

In any case, you must put `TrackProviderConsts.COLUMN_TRACK_WAVE` to the cursor as bytes (byte[] array), as Android cursor doesn't support other BLOB types.

Track wave `TrackProviderConsts.COLUMN_TRACK_WAVE` column is added by Poweramp to the projection columns during the 2nd phase of scanning (the metadata scan).


### Roots

For your Provider to be selectable in the Android system picker, the Roots should have `Root.FLAG_SUPPORTS_IS_CHILD` flag set. Also, `isChildDocument` method should be overridden and, at least,
it should return true or do a full documentId hierarchy check as needed.

See [ExampleProvider.java](src/main/java/com/maxmpz/powerampproviderexample/ExampleProvider.java#L188).

### Folders Hierarchy And User Selected Sub-folders

Poweramp Music Folders dialog allows sub-folders selection, where user may select some provider folder in the hierarchy under the appropriate root.
Such folder by default is shown on top level in the Folders Hierarchy category.

If you want such folders to be in the full (root/../subfolder) hierarchy despite user selection, implement
[TrackProviderConsts.CALL_GET_DIR_METADATA](../poweramp_api_lib/src/main/java/com/maxmpz/poweramp/player/TrackProviderConsts.java#L108).
See [ExampleProvider.java](src/main/java/com/maxmpz/powerampproviderexample/ExampleProvider.java#L951) for the example implementation.

Supported since 869.

### Sorting

Generally, categories sorting is set by user, but if sorting is set to the track number Poweramp looks into few sources of data:
- track meta information (MediaStore.Audio.AudioColumns.TRACK), if provided
- track tag, if no meta information provided
- track position in the cursor, if meta information is provided, but the track # is not specified (MediaStore.Audio.AudioColumns.TRACK is not defined in the cursor)

Since 870 Poweramp will apply optional TrackProviderConsts.COLUMN_TRACK_ALT track position in the cursor for Folders and Folders Hierarchy tracks
(for the "By Track #" user selected sorting only).
This is to allow provider flexible ordering control in these categories.
Other categories, such as Album tracks will continue to use ordering by track number tag.
TrackProviderConsts.COLUMN_TRACK_ALT is accepted for the queryChildDocuments returned cursors only and ignored e.g. in queryDocument.

For providers, Poweramp doesn't infer track number from the track filename as it does for filesystem based tracks (e.g. 01-track.mp3).

For folders in Folder Hierarchy, Poweramp always uses folder cursor position (build 869+). For flat Folders category sorting is set by user.

### Playlists
Your provider can provide m3u8/pls playlists with either http streams or links to the other tracks, including tracks provided by the provider.
Paths in the playlist should follow "parent_folder/file_name.ext" pattern.  
Poweramp uses just the (last) parent folder name and file name to resolve a playlist entry to the actual track/file.
The folder and file names come from Document.COLUMN_DISPLAY_NAME (for the appropriate folder or file).
Note that if you don't provide COLUMN_DISPLAY_NAME, Poweramp will try to generate some path based on doc id, but that won't be stable,
so using COLUMN_DISPLAY_NAME is recommended.
See [ExampleProvider](src/main/java/com/maxmpz/powerampproviderexample/ExampleProvider.java#L411) for details on filling the playlist cursor rows.

### Cue Sheets

At this moment, .cue sheet files are not support for the track providers. *Please create issue on github if this functionality is needed for your project.*


### Data Refresh

Data is refreshed on app first start, or when appropriate event or intent received
(see [Scanner intents in PowerampAPI.java](../poweramp_api_lib/src/main/java/com/maxmpz/poweramp/player/PowerampAPI.java#L1560)), etc.

Poweramp handles metadata refresh automatically based on the `Document.COLUMN_LAST_MODIFIED` column for the folders and files.

This is configurable in the Poweramp settings: for example, only manual rescan may work depending on the settings.

Please note that the Scanner intents sent to Poweramp will be only processed if your app is on the foreground, or if Poweramp is a foreground process (either Poweramp UI is visible or Poweramp is playing).  
If this is not true, scan service is a subject to the Android 8+ background services policy and your intent will be ignored.

Use [EXTRA_PROVIDER from PowerampAPI.java](../poweramp_api_lib/src/main/java/com/maxmpz/poweramp/player/PowerampAPI.java#L1694) for fine-grained scan just for your provider. If not specified,
Poweramp will scan all known folders and providers.

### Rescan
 
Poweramp optimizes scanning the providers to reduce possible hit on user battery (as any media file related change will trigger some sort
of rescan). By default Poweramp will scan Provider initially and won't rescan it automatically, unless:

- provider explicitly sends rescan request intent to Poweramp
- user uses Rescan option from within Provider's folder in Poweramp (Folders or Folders Hierarchy categories). This way only the given folder hierachy is rescanned
- user uses Rescan or Full Rescan option from Poweramp Settings / Library

There is also "Scan Providers" option (disabled by default) which will force Poweramp to rescan Providers automatically in other cases.

### Deletion

Poweramp will issue standard delete call when the user tries to delete one or multiple files. Either delete the track or throw an appropriate "not supported" exception to ignore deletion.


### Provider Crashes

Android may close client app "connected" to your Provider if your Provider process crashes.  
That means even minor exception in the Provider may cause complete unexpected Poweramp shutdown.
To avoid that, wrap your Provider public methods with `try/catch(Throwable)` with the appropriate logging.
Note that you still need to throw specific API-defined checked exceptions, such as `FileNotFoundException` from `openDocumentThumbnail`.


### Music Folders Selection Optimization

Poweramp adds small optimization for Music Folders selection dialog, where user can choose folders from filesystem or various providers.
To avoid loading all the provider possible folders (even assuming just first level folders loaded), Poweramp looks into additional custom
`TrackProviderConsts.COLUMN_FLAGS` column (note, that is NOT `Document.COLUMN_FLAGS`), which can be filled either with `TrackProviderConsts.FLAG_HAS_SUBDIRS` or `TrackProviderConsts.HAS_NO_SUBDIRS`.

If one or another flag is set for given folder (filled by provider in `queryDocuments` or `queryChildrenDocuments`), Poweramp won't further load anything
until user expands folders tree.

The TrackProviderConsts.COLUMN_FLAGS should be supported for both `queryDocument` (including root folders) and `queryChildrenDocuments`.
