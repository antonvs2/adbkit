# adbkit

**adbkit** is a pure [Node.js][nodejs] client for the [Android Debug Bridge][adb-site] server. It can be used either as a library in your own application, or simply as a convenient utility for playing with your device.

Most of the `adb` command line tool's functionality is supported (including pushing/pulling files, installing APKs and processing logs), with some added functionality such as being able to generate touch/key events and take screenshots. Some shims are provided for older devices, but we have not and will not test anything below Android 2.3.

Internally, we use this library to drive a multitude of Android devices from a variety of manufacturers, so we can say with a fairly high degree of confidence that it will most likely work with your device(s), too.

# Incompatible changes since 1.x.x

Previously, we made extensive use of callbacks in almost every feature. While this normally works okay, ADB connections can be quite fickle, and it was starting to become difficult to handle every possible error. For example, we'd often fail to properly clean up after ourselves when a connection suddenly died in an unexpected place, causing memory and resource leaks.

In version 2, we've replaced nearly all callbacks with [Promises](http://promisesaplus.com/) (using [Bluebird](https://github.com/petkaantonov/bluebird)), allowing for much more reliable error propagation and resource cleanup (thanks to `.finally()`). Additionally, many commands can now be cancelled on the fly, and although unimplemented at this point, we'll also be able to report progress on long-running commands without any changes to the API.

Unfortunately, some API changes were required for this change. `client.framebuffer()`'s callback, for example, previously accepted more than one argument, which doesn't translate into Promises so well. Thankfully, it made sense to combine the arguments anyway, and we were able to do it quite cleanly.

Furthermore, most API methods were returning the current instance for chaining purposes. While perhaps useful in some contexts, most of the time it probably didn't quite do what users expected, as chained calls were run in parallel rather than in serial fashion. Now every applicable API method returns a Promise, which is an incompatible but welcome change. This will also allow you to hook into `yield` and coroutines in Node 0.12.

**However, all methods still accept (and will accept in the future) callbacks for those who prefer them.**

Test coverage was also massively improved, although we've still got ways to go.

## Requirements

* [Node.js][nodejs] >= 0.10
* The `adb` command line tool

Please note that although it may happen at some point, **this project is NOT an implementation of the ADB _server_**. The target host (where the devices are connected) must still have ADB installed and either already running (e.g. via `adb start-server`) or available in `$PATH`. An attempt will be made to start the server locally via the aforementioned command if the initial connection fails. This is the only case where we fall back to the `adb` binary.

When targeting a remote host, starting the server is entirely your responsibility.

Alternatively, you may want to consider using the Chrome [ADB][chrome-adb] extension, as it includes the ADB server and can be started/stopped quite easily.

## Getting started

Install via NPM:

```bash
npm install --save adbkit
```

We use [debug][node-debug], and our debug namespace is `adb`. Some of the dependencies may provide debug output of their own. To see the debug output, set the `DEBUG` environment variable. For example, run your program with `DEBUG=adb:* node app.js`.

Note that even though the module is written in [CoffeeScript][coffeescript], only the compiled JavaScript is published to [NPM][npm], which means that it can easily be used with pure JavaScript codebases, too.

### Examples

The examples may be a bit verbose, but that's because we're trying to keep them as close to real-life code as possible, with flow control and error handling taken care of.

#### Checking for NFC support

```js
var Promise = require('bluebird')
var adb = require('adbkit')
var client = adb.createClient()

client.listDevices()
  .then(function(devices) {
    return Promise.filter(devices, function(device) {
      return client.getFeatures(device.id)
        .then(function(features) {
          return features['android.hardware.nfc']
        })
    })
  })
  .then(function(supportedDevices) {
    console.log('The following devices support NFC:', supportedDevices)
  })
  .catch(function(err) {
    console.error('Something went wrong:', err.stack)
  })
```

#### Installing an APK

```js
var Promise = require('bluebird')
var adb = require('adbkit')
var client = adb.createClient()
var apk = 'vendor/app.apk'

client.listDevices()
  .then(function(devices) {
    return Promise.map(devices, function(device) {
      return client.install(device.id, apk)
    })
  })
  .then(function() {
    console.log('Installed %s on all connected devices', apk)
  })
  .catch(function(err) {
    console.error('Something went wrong:', err.stack)
  })
```

#### Tracking devices

```js
var adb = require('adbkit')
var client = adb.createClient()

client.trackDevices()
  .then(function(tracker) {
    tracker.on('add', function(device) {
      console.log('Device %s was plugged in', device.id)
    })
    tracker.on('remove', function(device) {
      console.log('Device %s was unplugged', device.id)
    })
    tracker.on('end', function() {
      console.log('Tracking stopped')
    })
  })
  .catch(function(err) {
    console.error('Something went wrong:', err.stack)
  })
```

#### Pulling a file from all connected devices

```js
var Promise = require('bluebird')
var fs = require('fs')
var adb = require('adbkit')
var client = adb.createClient()

client.listDevices()
  .then(function(devices) {
    return Promise.map(devices, function(device) {
      return client.pull(device.id, '/system/build.prop')
        .then(function(transfer) {
          return new Promise(function(resolve, reject) {
            var fn = '/tmp/' + device.id + '.build.prop'
            transfer.on('progress', function(stats) {
              console.log('[%s] Pulled %d bytes so far',
                device.id,
                stats.bytesTransferred)
            })
            transfer.on('end', function() {
              console.log('[%s] Pull complete', device.id)
              resolve(device.id)
            })
            transfer.on('error', reject)
            transfer.pipe(fs.createWriteStream(fn))
          })
        })
    })
  })
  .then(function() {
    console.log('Done pulling /system/build.prop from all connected devices')
  })
  .catch(function(err) {
    console.error('Something went wrong:', err.stack)
  })
```

#### Pushing a file to all connected devices

```js
var Promise = require('bluebird')
var adb = require('adbkit')
var client = adb.createClient()

client.listDevices()
  .then(function(devices) {
    return Promise.map(devices, function(device) {
      return client.push(device.id, 'temp/foo.txt', '/data/local/tmp/foo.txt')
        .then(function(transfer) {
          return new Promise(function(resolve, reject) {
            transfer.on('progress', function(stats) {
              console.log('[%s] Pushed %d bytes so far',
                device.id,
                stats.bytesTransferred)
            })
            transfer.on('end', function() {
              console.log('[%s] Push complete', device.id)
              resolve()
            })
            transfer.on('error', reject)
          })
        })
    })
  })
  .then(function() {
    console.log('Done pushing foo.txt to all connected devices')
  })
  .catch(function(err) {
    console.error('Something went wrong:', err.stack)
  })
```

#### List files in a folder

```js
var Promise = require('bluebird')
var adb = require('adbkit')
var client = adb.createClient()

client.listDevices()
  .then(function(devices) {
    return Promise.map(devices, function(device) {
      return client.readdir(device.id, '/sdcard')
        .then(function(files) {
          // Synchronous, so we don't have to care about returning at the
          // right time
          files.forEach(function(file) {
            if (file.isFile()) {
              console.log('[%s] Found file "%s"', device.id, file.name)
            }
          })
        })
    })
  })
  .then(function() {
    console.log('Done checking /sdcard files on connected devices')
  })
  .catch(function(err) {
    console.error('Something went wrong:', err.stack)
  })
```

## API

### ADB

#### adb.createClient([options])

Creates a client instance with the provided options. Note that this will not automatically establish a connection, it will only be done when necessary.

* **options** An object compatible with [Net.connect][net-connect]'s options:
    - **port** The port where the ADB server is listening. Defaults to `5037`.
    - **host** The host of the ADB server. Defaults to `'localhost'`.
    - **bin** As the sole exception, this option provides the path to the `adb` binary, used for starting the server locally if initial connection fails. Defaults to `'adb'`.
* Returns: The client instance.

### Client

#### client.clear(serial, pkg[, callback])

Deletes all data associated with a package from the device. This is roughly analogous to `adb shell pm clear <pkg>`.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **pkg** The package name. This is NOT the APK.
* **callback(err)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
* Returns: `Promise`
* Resolves with: `true`

#### client.forward(serial, local, remote[, callback])

Forwards socket connections from the ADB server host (local) to the device (remote). This is analogous to `adb forward <local> <remote>`. It's important to note that if you are connected to a remote ADB server, the forward will be created on that host.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **local** A string representing the local endpoint on the ADB host. At time of writing, can be one of:
    - `tcp:<port>`
    - `localabstract:<unix domain socket name>`
    - `localreserved:<unix domain socket name>`
    - `localfilesystem:<unix domain socket name>`
    - `dev:<character device name>`
* **remote** A string representing the remote endpoint on the device. At time of writing, can be one of:
    - Any value accepted by the `local` argument
    - `jdwp:<process pid>`
* **callback(err)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
* Returns: `Promise`
* Resolves with: `true`

#### client.framebuffer(serial[, format]&#91;, callback])

Fetches the current **raw** framebuffer (i.e. what is visible on the screen) from the device, and optionally converts it into something more usable by using [GraphicsMagick][graphicsmagick]'s `gm` command, which must be available in `$PATH` if conversion is desired. Note that we don't bother supporting really old framebuffer formats such as RGB_565. If for some mysterious reason you happen to run into a `>=2.3` device that uses RGB_565, let us know.

Note that high-resolution devices can have quite massive framebuffers. For example, a device with a resolution of 1920x1080 and 32 bit colors would have a roughly 8MB (`1920*1080*4` byte) RGBA framebuffer. Empirical tests point to about 5MB/s bandwidth limit for the ADB USB connection, which means that it can take ~1.6 seconds for the raw data to arrive, or even more if the USB connection is already congested. Using a conversion will further slow down completion.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **format** The desired output format. Any output format supported by [GraphicsMagick][graphicsmagick] (such as `'png'`) is supported. Defaults to `'raw'` for raw framebuffer data.
* **callback(err, framebuffer)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **framebuffer** The possibly converted framebuffer stream. The stream also has a `meta` property with the following values:
        * **version** The framebuffer version. Useful for patching possible backwards-compatibility issues.
        * **bpp** Bits per pixel (i.e. color depth).
        * **size** The raw byte size of the framebuffer.
        * **width** The horizontal resolution of the framebuffer. This SHOULD always be the same as screen width. We have not encountered any device with incorrect framebuffer metadata, but according to rumors there might be some.
        * **height** The vertical resolution of the framebuffer. This SHOULD always be the same as screen height.
        * **red_offset** The bit offset of the red color in a pixel.
        * **red_length** The bit length of the red color in a pixel.
        * **blue_offset** The bit offset of the blue color in a pixel.
        * **blue_length** The bit length of the blue color in a pixel.
        * **green_offset** The bit offset of the green color in a pixel.
        * **green_length** The bit length of the green color in a pixel.
        * **alpha_offset** The bit offset of alpha in a pixel.
        * **alpha_length** The bit length of alpha in a pixel. `0` when not available.
        * **format** The framebuffer format for convenience. This can be one of `'bgr'`,  `'bgra'`, `'rgb'`, `'rgba'`.
* Returns: `Promise`
* Resolves with: `framebuffer` (see callback)

#### client.getDevicePath(serial[, callback])

Gets the device path of the device identified by the given serial number.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, path)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **path** The device path. This corresponds to the device path in `client.listDevicesWithPaths()`.
* Returns: `Promise`
* Resolves with: `path` (see callback)

#### client.getFeatures(serial[, callback])

Retrieves the features of the device identified by the given serial number. This is analogous to `adb shell pm list features`. Useful for checking whether hardware features such as NFC are available (you'd check for `'android.hardware.nfc'`).

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, features)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **features** An object of device features. Each key corresponds to a device feature, with the value being either `true` for a boolean feature, or the feature value as a string (e.g. `'0x20000'` for `reqGlEsVersion`).
* Returns: `Promise`
* Resolves with: `features` (see callback)

#### client.getPackages(serial[, callback])

Retrieves the list of packages present on the device. This is analogous to `adb shell pm list packages`. If you just want to see if something's installed, consider using `client.isInstalled()` instead.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, packages)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **packages** An array of package names.
* Returns: `Promise`
* Resolves with: `packages` (see callback)

#### client.getProperties(serial[, callback])

Retrieves the properties of the device identified by the given serial number. This is analogous to `adb shell getprop`.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, properties)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **properties** An object of device properties. Each key corresponds to a device property. Convenient for accessing things like `'ro.product.model'`.
* Returns: `Promise`
* Resolves with: `properties` (see callback)

#### client.getSerialNo(serial[, callback])

Gets the serial number of the device identified by the given serial number. With our API this doesn't really make much sense, but it has been implemented for completeness. _FYI: in the raw ADB protocol you can specify a device in other ways, too._

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, serial)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **serial** The serial number of the device.
* Returns: `Promise`
* Resolves with: `serial` (see callback)

#### client.getState(serial[, callback])

Gets the state of the device identified by the given serial number.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, state)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **state** The device state. This corresponds to the device type in `client.listDevices()`.
* Returns: `Promise`
* Resolves with: `state` (see callback)

#### client.install(serial, apk[, callback])

Installs the APK on the device, replacing any previously installed version. This is roughly analogous to `adb install -r <apk>`.

Note that if the call seems to stall, you may have to accept a dialog on the phone first.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **apk** When `String`, interpreted as a path to an APK file. When [`Stream`][node-stream], installs directly from the stream, which must be a valid APK.
* **callback(err)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
* Returns: `Promise`
* Resolves with: `true`

#### client.installRemote(serial, apk[, callback])

Installs an APK file which must already be located on the device file system, and replaces any previously installed version. Useful if you've previously pushed the file to the device for some reason (perhaps to have direct access to `client.push()`'s transfer stats). This is roughly analogous to `adb shell pm install -r <apk>` followed by `adb shell rm -f <apk>`.

Note that if the call seems to stall, you may have to accept a dialog on the phone first.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **apk** The path to the APK file on the device. The file will be removed when the command completes.
* **callback(err)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
* Returns: `Promise`
* Resolves with: `true`

#### client.isInstalled(serial, pkg[, callback])

Tells you if the specific package is installed or not. This is analogous to `adb shell pm path <pkg>` and some output parsing.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **pkg** The package name. This is NOT the APK.
* **callback(err, installed)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **installed** `true` if the package is installed, `false` otherwise.
* Returns: `Promise`
* Resolves with: `installed` (see callback)

#### client.kill([callback])

This kills the ADB server. Note that the next connection will attempt to start the server again when it's unable to connect.

* **callback(err)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
* Returns: `Promise`
* Resolves with: `true`

#### client.listDevices([callback])

Gets the list of currently connected devices and emulators.

* **callback(err, devices)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **devices** An array of device objects. The device objects are plain JavaScript objects with two properties: `id` and `type`.
        * **id** The ID of the device. For real devices, this is usually the USB identifier.
        * **type** The device type. Values include `'emulator'` for emulators, `'device'` for devices, and `'offline'` for offline devices. `'offline'` can occur for example during boot, in low-battery conditions or when the ADB connection has not yet been approved on the device.
* Returns: `Promise`
* Resolves with: `devices` (see callback)

#### client.listDevicesWithPaths([callback])

Like `client.listDevices()`, but includes the "path" of every device.

* **callback(err, devices)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **devices** An array of device objects. The device objects are plain JavaScript objects with the following properties:
        * **id** See `client.listDevices()`.
        * **type** See `client.listDevices()`.
        * **path** The device path. This can be something like `usb:FD120000` for real devices.
* Returns: `Promise`
* Resolves with: `devices` (see callback)

#### client.listForwards(serial[, callback])

Lists forwarded connections on the device. This is analogous to `adb forward --list`.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, forwards)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **forwards** An array of forward objects with the following properties:
        * **serial** The device serial.
        * **local** The local endpoint. Same format as `client.forward()`'s `local` argument.
        * **remote** The remote endpoint on the device. Same format as `client.forward()`'s `remote` argument.
* Returns: `Promise`
* Resolves with: `forwards` (see callback)

#### client.openLog(serial, name[, callback])

Opens a direct connection to a binary log file, providing access to the raw log data. Note that it is usually much more convenient to use the `client.openLogcat()` method, described separately.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **name** The name of the log. Available logs include `'main'`, `'system'`, `'radio'` and `'events'`.
* **callback(err, log)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **log** The binary log stream. Call `log.end()` when you wish to stop receiving data.
* Returns: `Promise`
* Resolves with: `log` (see callback)

#### client.openLogcat(serial[, options]&#91;, callback])

Calls the `logcat` utility on the device and hands off the connection to [adbkit-logcat][adbkit-logcat], a pure Node.js Logcat client. This is analogous to `adb logcat -B`, but the event stream will be parsed for you and a separate event will be emitted for every log entry, allowing for easy processing.

For more information, check out the [adbkit-logcat][adbkit-logcat] documentation.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **options** Optional. The following options are supported:
    - **clear** When `true`, clears logcat before opening the reader. Not set by default.
* **callback(err, logcat)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **logcat** The Logcat client. Please see the [adbkit-logcat][adbkit-logcat] documentation for details.
* Returns: `Promise`
* Resolves with: `logcat` (see callback)

#### client.openMonkey(serial[, port]&#91;, callback])

Starts the built-in `monkey` utility on the device, connects to it using `client.openTcp()` and hands the connection to [adbkit-monkey][adbkit-monkey], a pure Node.js Monkey client. This allows you to create touch and key events, among other things.

For more information, check out the [adbkit-monkey][adbkit-monkey] documentation.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **port** Optional. The device port where you'd like Monkey to run at. Defaults to `1080`.
* **callback(err, monkey)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **monkey** The Monkey client. Please see the [adbkit-monkey][adbkit-monkey] documentation for details.
* Returns: `Promise`
* Resolves with: `monkey` (see callback)

#### client.openProcStat(serial[, callback])

Tracks `/proc/stat` and emits useful information, such as CPU load. A single sync service instance is used to download the `/proc/stat` file for processing. While doing this does consume some resources, it is very light and should not be a problem.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, stats)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **stats** The `/proc/stat` tracker, which is an [`EventEmitter`][node-events]. Call `stat.end()` to stop tracking. The following events are available:
        * **load** **(loads)** Emitted when a CPU load calculation is available.
            - **loads** CPU loads of **online** CPUs. Each key is a CPU id (e.g. `'cpu0'`, `'cpu1'`) and the value an object with the following properties:
                * **user** Percentage (0-100) of ticks spent on user programs.
                * **nice** Percentage (0-100) of ticks spent on `nice`d user programs.
                * **system** Percentage (0-100) of ticks spent on system programs.
                * **idle** Percentage (0-100) of ticks spent idling.
                * **iowait** Percentage (0-100) of ticks spent waiting for IO.
                * **irq** Percentage (0-100) of ticks spent on hardware interrupts.
                * **softirq** Percentage (0-100) of ticks spent on software interrupts.
                * **steal** Percentage (0-100) of ticks stolen by others.
                * **guest** Percentage (0-100) of ticks spent by a guest.
                * **guestnice** Percentage (0-100) of ticks spent by a `nice`d guest.
                * **total** Total. Always 100.
* Returns: `Promise`
* Resolves with: `stats` (see callback)

#### client.openTcp(serial, port[, host]&#91;, callback])

Opens a direct TCP connection to a port on the device, without any port forwarding required.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **port** The port number to connect to.
* **host** Optional. The host to connect to. Allegedly this is supposed to establish a connection to the given host from the device, but we have not been able to get it to work at all. Skip the host and everything works great.
* **callback(err, conn)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **conn** The TCP connection (i.e. [`net.Socket`][node-net]). Read and write as you please. Call `conn.end()` to end the connection.
* Returns: `Promise`
* Resolves with: `conn` (see callback)

#### client.pull(serial, path[, callback])

A convenience shortcut for `sync.pull()`, mainly for one-off use cases. The connection cannot be reused, resulting in poorer performance over multiple calls. However, the Sync client will be closed automatically for you, so that's one less thing to worry about.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **path** See `sync.pull()` for details.
* **callback(err, transfer)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **transfer** A `PullTransfer` instance (see below)
* Returns: `Promise`
* Resolves with: `transfer` (see callback)

#### client.push(serial, contents, path[, mode]&#91;, callback])

A convenience shortcut for `sync.push()`, mainly for one-off use cases. The connection cannot be reused, resulting in poorer performance over multiple calls. However, the Sync client will be closed automatically for you, so that's one less thing to worry about.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **contents** See `sync.push()` for details.
* **path** See `sync.push()` for details.
* **mode** See `sync.push()` for details.
* **callback(err, transfer)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **transfer** A `PushTransfer` instance (see below)
* Returns: `Promise`
* Resolves with: `transfer` (see callback)

#### client.readdir(serial, path[, callback])

A convenience shortcut for `sync.readdir()`, mainly for one-off use cases. The connection cannot be reused, resulting in poorer performance over multiple calls. However, the Sync client will be closed automatically for you, so that's one less thing to worry about.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **path** See `sync.readdir()` for details.
* **callback(err, files)** Optional. Use this or the returned `Promise`. See `sync.readdir()` for details.
* Returns: `Promise`
* Resolves with: See `sync.readdir()` for details.

#### client.remount(serial[, callback])

Attempts to remount the `/system` partition in read-write mode. This will usually only work on emulators and developer devices.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
* Returns: `Promise`
* Resolves with: `true`

#### client.screencap(serial[, callback])

Takes a screenshot in PNG format using the built-in `screencap` utility. This is analogous to `adb shell screencap -p`. Sadly, the utility is not available on most Android `<=2.3` devices, but a silent fallback to the `client.framebuffer()` command in PNG mode is attempted, so you should have its dependencies installed just in case.

Generating the PNG on the device naturally requires considerably more processing time on that side. However, as the data transferred over USB easily decreases by ~95%, and no conversion being required on the host, this method is usually several times faster than using the framebuffer. Naturally, this benefit does not apply if we're forced to fall back to the framebuffer.

For convenience purposes, if the screencap command fails (e.g. because it doesn't exist on older Androids), we fall back to `client.framebuffer(serial, 'png')`, which is slower and has additional installation requirements.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, screencap)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **screencap** The PNG stream.
* Returns: `Promise`
* Resolves with: `screencap` (see callback)

#### client.shell(serial, command[, callback])

Runs a shell command on the device. Note that you'll be limited to the permissions of the `shell` user, which ADB uses.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **command** The shell command to execute. When `String`, the command is run as-is. When `Array`, the elements will be rudimentarily escaped (for convenience, not security) and joined to form a command.
* **callback(err, output)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **output** An output [`Stream`][node-stream] in non-flowing mode. Unfortunately it is not possible to separate stdout and stderr, you'll get both of them in one stream. It is also not possible to access the exit code of the command. If access to any of these individual properties is needed, the command must be constructed in a way that allows you to parse the information from the output.
* Returns: `Promise`
* Resolves with: `output` (see callback)

#### client.startActivity(serial, options[, callback])

Starts the configured activity on the device. Roughly analogous to `adb shell am start <options>`.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **options** The activity configuration. The following options are available:
    - **action** The action.
    - **data** The data URI, if any.
    - **mimeType** The mime type, if any.
    - **category** The category. For multiple categories, pass an `Array`.
    - **component** The component.
    - **flags** Numeric flags.
    - **extras** Any extra data.
        * When an `Array`, each item must be an `Object` the following properties:
            - **key** The key name.
            - **type** The type, which can be one of `'string'`, `'null'`, `'bool'`, `'int'`, `'long'`, `'float'`, `'uri'`, `'component'`.
            - **value** The value. Optional and unused if type is `'null'`. If an `Array`, type is automatically set to be an array of `<type>`.
        * When an `Object`, each key is treated as the key name. Simple values like `null`, `String`, `Boolean` and `Number` are type-mapped automatically (`Number` maps to `'int'`) and can be used as-is. For more complex types, like arrays and URIs, set the value to be an `Object` like in the Array syntax (see above), but leave out the `key` property.
* **callback(err)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
* Returns: `Promise`
* Resolves with: `true`

#### client.stat(serial, path[, callback])

A convenience shortcut for `sync.stat()`, mainly for one-off use cases. The connection cannot be reused, resulting in poorer performance over multiple calls. However, the Sync client will be closed automatically for you, so that's one less thing to worry about.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **path** See `sync.stat()` for details.
* **callback(err, stats)** Optional. Use this or the returned `Promise`. See `sync.stat()` for details.
* Returns: `Promise`
* Resolves with: See `sync.stat()` for details.

#### client.syncService(serial[, callback])

Establishes a new Sync connection that can be used to push and pull files. This method provides the most freedom and the best performance for repeated use, but can be a bit cumbersome to use. For simple use cases, consider using `client.stat()`, `client.push()` and `client.pull()`.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, sync)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **sync** The Sync client. See below for details. Call `sync.end()` when done.
* Returns: `Promise`
* Resolves with: `sync` (see callback)

#### client.trackDevices([callback])

Gets a device tracker. Events will be emitted when devices are added, removed, or their type changes (i.e. to/from `offline`). Note that the same events will be emitted for the initially connected devices also, so that you don't need to use both `client.listDevices()` and `client.trackDevices()`.

Note that as the tracker will keep a connection open, you must call `tracker.end()` if you wish to stop tracking devices.

* **callback(err, tracker)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **tracker** The device tracker, which is an [`EventEmitter`][node-events]. The following events are available:
        * **add** **(device)** Emitted when a new device is connected, once per device. See `client.listDevices()` for details on the device object.
        * **remove** **(device)** Emitted when a device is unplugged, once per device. This does not include `offline` devices, those devices are connected but unavailable to ADB. See `client.listDevices()` for details on the device object.
        * **change** **(device)** Emitted when the `type` property of a device changes, once per device. The current value of `type` is the new value. This event usually occurs the type changes from `'device'` to `'offline'` or the other way around. See `client.listDevices()` for details on the device object and the `'offline'` type.
        * **changeSet** **(changes)** Emitted once for all changes reported by ADB in a single run. Multiple changes can occur when, for example, a USB hub is connected/unplugged and the device list changes quickly. If you wish to process all changes at once, use this event instead of the once-per-device ones. Keep in mind that the other events will still be emitted, though.
            - **changes** An object with the following properties always present:
                * **added** An array of added device objects, each one as in the `add` event. Empty if none.
                * **removed** An array of removed device objects, each one as in the `remove` event. Empty if none.
                * **changed** An array of changed device objects, each one as in the `change` event. Empty if none.
        * **end** Emitted when the underlying connection ends.
        * **error** **(err)** Emitted if there's an error.
* Returns: `Promise`
* Resolves with: `tracker` (see callback)

#### client.uninstall(serial, pkg[, callback])

Uninstalls the package from the device. This is roughly analogous to `adb uninstall <pkg>`.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **pkg** The package name. This is NOT the APK.
* **callback(err)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
* Returns: `Promise`
* Resolves with: `true`

#### client.version([callback])

Queries the ADB server for its version. This is mainly useful for backwards-compatibility purposes.

* **callback(err, version)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **version** The version of the ADB server.
* Returns: `Promise`
* Resolves with: `version` (see callback)

#### client.waitBootComplete(serial[, callback])

Waits until the device has finished booting. Note that the device must already be seen by ADB. This is roughly analogous to periodically checking `adb shell getprop sys.boot_completed`.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err)** Optional. Use this or the returned `Promise`.
    - **err** `null` if the device has completed booting, `Error` otherwise (can occur if the connection dies while checking).
* Returns: `Promise`
* Resolves with: `true`

### Sync

#### sync.end()

Closes the Sync connection, allowing Node to quit (assuming nothing else is keeping it alive, of course).

* Returns: The sync instance.

#### sync.pull(path)

Pulls a file from the device as a `PullTransfer` [`Stream`][node-stream].

* **path** The path to pull from.
* Returns: A `PullTransfer` instance. See below for details.

#### sync.push(contents, path[, mode])

Attempts to identify `contents` and calls the appropriate `push*` method for it.

* **contents** When `String`, treated as a local file path and forwarded to `sync.pushFile()`. Otherwise, treated as a [`Stream`][node-stream] and forwarded to `sync.pushStream()`.
* **path** The path to push to.
* **mode** Optional. The mode of the file. Defaults to `0644`.
* Returns: A `PushTransfer` instance. See below for details.

#### sync.pushFile(file, path[, mode])

Pushes a local file to the given path. Note that the path must be writable by the ADB user (usually `shell`). When in doubt, use `'/data/local/tmp'` with an appropriate filename.

* **file** The local file path.
* **path** See `sync.push()` for details.
* **mode** See `sync.push()` for details.
* Returns: See `sync.push()` for details.

#### sync.pushStream(stream, path[, mode])

Pushes a [`Stream`][node-stream] to the given path. Note that the path must be writable by the ADB user (usually `shell`). When in doubt, use `'/data/local/tmp'` with an appropriate filename.

* **stream** The readable stream.
* **path** See `sync.push()` for details.
* **mode** See `sync.push()` for details.
* Returns: See `sync.push()` for details.

#### sync.readdir(path[, callback])

Retrieves a list of directory entries (e.g. files) in the given path, not including the `.` and `..` entries, just like [`fs.readdir`][node-fs]. If given a non-directory path, no entries are returned.

* **path** The path.
* **callback(err, files)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **files** An `Array` of [`fs.Stats`][node-fs-stats]-compatible instances. While the `stats.is*` methods are available, only the following properties are supported (in addition to the `name` field which contains the filename):
        * **name** The filename.
        * **mode** The raw mode.
        * **size** The file size.
        * **mtime** The time of last modification as a `Date`.
* Returns: `Promise`
* Resolves with: `files` (see callback)

#### sync.stat(path[, callback])

Retrieves information about the given path.

* **path** The path.
* **callback(err, stats)** Optional. Use this or the returned `Promise`.
    - **err** `null` when successful, `Error` otherwise.
    - **stats** An [`fs.Stats`][node-fs-stats] instance. While the `stats.is*` methods are available, only the following properties are supported:
        * **mode** The raw mode.
        * **size** The file size.
        * **mtime** The time of last modification as a `Date`.
* Returns: `Promise`
* Resolves with: `stats` (see callback)

#### sync.tempFile(path)

A simple helper method for creating appropriate temporary filenames for pushing files. This is essentially the same as taking the basename of the file and appending it to `'/data/local/tmp/'`.

* **path** The path of the file.
* Returns: An appropriate temporary file path.

### PushTransfer

A simple EventEmitter, mainly for keeping track of the progress.

List of events:

* **progress** **(stats)** Emitted when a chunk has been flushed to the ADB connection.
    - **stats** An object with the following stats about the transfer:
        * **bytesTransferred** The number of bytes transferred so far.
* **error** **(err)** Emitted on error.
    - **err** An `Error`.
* **end** Emitted when the transfer has successfully completed.

#### pushTransfer.cancel()

Cancels the transfer by ending both the stream that is being pushed and the sync connection. This will most likely end up creating a broken file on your device. **Use at your own risk.** Also note that you must create a new sync connection if you wish to continue using the sync service.

* Returns: The pushTransfer instance.

### PullTransfer

`PullTransfer` is a [`Stream`][node-stream]. Use [`fs.createWriteStream()`][node-fs] to pipe the stream to a file if necessary.

List of events:

* **progress** **(stats)** Emitted when a new chunk is received.
    - **stats** An object with the following stats about the transfer:
        * **bytesTransferred** The number of bytes transferred so far.
* **error** **(err)** Emitted on error.
    - **err** An `Error`.
* **end** Emitted when the transfer has successfully completed.

#### pullTransfer.cancel()

Cancels the transfer by ending the connection. Can be useful for reading endless streams of data, such as `/dev/urandom` or `/dev/zero`, perhaps for benchmarking use. Note that you must create a new sync connection if you wish to continue using the sync service.

* Returns: The pullTransfer instance.

## More information

* [Android Debug Bridge][adb-site]
    - [SERVICES.TXT][adb-services] (ADB socket protocol)
* [Android ADB Protocols][adb-protocols] (a blog post explaining the protocol)
* [adb.js][adb-js] (another Node.js ADB implementation)
* [ADB Chrome extension][chrome-adb]

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

See [LICENSE](LICENSE).

Copyright © CyberAgent, Inc. All Rights Reserved.

[nodejs]: <http://nodejs.org/>
[coffeescript]: <http://coffeescript.org/>
[npm]: <https://npmjs.org/>
[adb-js]: <https://github.com/flier/adb.js>
[adb-site]: <http://developer.android.com/tools/help/adb.html>
[adb-services]: <https://github.com/android/platform_system_core/blob/master/adb/SERVICES.TXT>
[adb-protocols]: <http://blogs.kgsoft.co.uk/2013_03_15_prg.htm>
[file_sync_service.h]: <https://github.com/android/platform_system_core/blob/master/adb/file_sync_service.h>
[chrome-adb]: <https://chrome.google.com/webstore/detail/adb/dpngiggdglpdnjdoaefidgiigpemgage>
[node-debug]: <https://npmjs.org/package/debug>
[net-connect]: <http://nodejs.org/api/net.html#net_net_connect_options_connectionlistener>
[node-events]: <http://nodejs.org/api/events.html>
[node-stream]: <http://nodejs.org/api/stream.html>
[node-net]: <http://nodejs.org/api/net.html>
[node-fs]: <http://nodejs.org/api/fs.html>
[node-fs-stats]: <http://nodejs.org/api/fs.html#fs_class_fs_stats>
[node-gm]: <https://github.com/aheckmann/gm>
[graphicsmagick]: <http://www.graphicsmagick.org/>
[imagemagick]: <http://www.imagemagick.org/>
[adbkit-logcat]: <https://npmjs.org/package/adbkit-logcat>
[adbkit-monkey]: <https://npmjs.org/package/adbkit-monkey>
