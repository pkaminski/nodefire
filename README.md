NodeFire
========

[![Project Status: Active - The project has reached a stable, usable state and is being actively developed.](http://www.repostatus.org/badges/latest/active.svg)](http://www.repostatus.org/#active)

NodeFire is a Firebase library for NodeJS that has a pretty similar API but adds the following features:

1. Most methods return a promise, instead of requiring a callback.  This works especially nicely with a framework like `co`, allowing you to control data flow with `yield` statements.
2. Any paths passed in are treated as templates and interpolated within an implicit or explicit scope, avoiding manual (and error-prone) string concatenation.  Characters forbidden by Firebase are automatically escaped.
3. Since one-time fetches of a reference are common in server code, a new `get` method makes them easy and an optional LRU cache keeps the most used ones pinned and synced to reduce latency.
4. Transactions prefetch the current value of the reference to avoid having every transaction re-executed at least twice.
5. In debug mode, traces are printed for failed security rule checks (and only failed ones).

If you'd like to be able to use generators as `on` or `once` callbacks, make sure to set `Promise.on` to a `co`-compatible function.

## Example

```javascript
var co = require('co');
var NodeFire = require('nodefire');
NodeFire.setCacheSize(10);
NodeFire.DEBUG = true;
var db = new NodeFire('https://example.firebaseio.com/');
co((function*() {
  yield db.auth('secret', {uid: 'server01'});
  var stuff = db.child('stuffs', {foo: 'bar', baz: {qux: 42}});
  var data = yield {
    theFoo: stuff.child('foos/:foo').get(),
    theUser: stuff.root().child('users/{baz.qux}').get()
  };
  yield [
    stuff.child('bars/:foo/{theUser.username}', data).set(data.theFoo.bar),
    stuff.child('counters/:foo').transaction(function(value) {return value + 1;})
  ];
})());
```

## API

This is reproduced from the source code, which is authoritative.

```javascript
/**
 * A wrapper around a Firebase reference, and the main entry point to the module.  You can pretty
 * much use this class as you would use the Firebase class.  The two major differences are:
 * 1) Most methods return promises, which you use instead of callbacks.
 * 2) Each NodeFire object has a scope dictionary associated with it that's used to interpolate any
 *    path used in a call.
 *
 * Every method that returns a promise also accepts an options object as the last argument.  One
 * standard option is `timeout`, which will cause an operation to time out after the given number of
 * milliseconds.  Other operation-specific options are described in their respective doc comments.
 */
class NodeFire;

/**
 * Creates a new NodeFire wrapper around a raw Firebase reference.
 *
 * @param {string || Firebase} refOrUrl The Firebase URL (or Firebase reference instance) that this
 *     object will represent.
 * @param {Object} scope Optional dictionary that will be used for interpolating paths.
 * @param {string} host For internal use only, do not pass.
 */
constructor(refOrUrl, scope, host)

/**
 * Flag that indicates whether to log transactions and the number of tries needed.
 * @type {boolean} True to log metadata about every transaction.
 */
static LOG_TRANSACTIONS = false

/**
 * A special constant that you can pass to enableFirebaseLogging to keep a rolling buffer of
 * Firebase logs without polluting the console, to be grabbed and saved only when needed.
 */
static ROLLING

/* Some static methods copied over from the Firebase class. */
static goOffline()
static goOnline()
static ServerValue

/**
 * Turn Firebase low-level connection logging on or off.
 * @param {boolean | ROLLING} enable Whether to enable or disable logging, or enable rolling logs.
 *        Rolling logs can only be enabled once; any further calls to enableFirebaseLogging will
 *        disable them permanently.
 * @returns If ROLLING logs were requested, returns a handle to FirebaseRollingLog (see
 *          https://github.com/mikelehen/firebase-rolling-log for details).  Otherwise returns
 *          nothing.
 */
static enableFirebaseLogging(enable)

/**
 * Adds an intercepting callback before all NodeFire database operations.  This callback can
 * modify the operation's options or block it while performing other work.
 * @param {Function} callback The callback to invoke before each operation.  It will be passed two
 *     arguments: an operation descriptor ({ref, method, args}) and an options object.  The
 *     descriptor is read-only but the options can be modified.  The callback can return any value
 *     (which will be ignored) or a promise, to block execution of the operation (but not other
 *     interceptors) until the promise settles.
 */
static interceptOperations(callback)

/**
 * Sets the maximum number of values to keep pinned and updated in the cache.  The cache is not used
 * unless you set a non-zero maximum.
 * @param {number} max The maximum number of values to keep pinned in the cache.
 */
static setCacheSize(max)

/**
 * Sets the maximum number of pinned values to retain in the cache when a host gets disconnected.
 * By default all values are retained, but if your cache size is high they'll all need to be double-
 * checked against the Firebase server when the connection comes back.  It may thus be more
 * economical to drop the least used ones when disconnected.
 * @param {number} max The maximum number of values from a disconnected host to keep pinned in the
 *        cache.
 */
static setCacheSizeForDisconnectedHost(max)

/**
 * Gets the current number of values pinned in the cache.
 * @return {number} The current size of the cache.
 */
static getCacheCount()

/**
 * Gets the current cache hit rate.  This is very approximate, as it's only counted for get() and
 * transaction() calls, and is unable to count ancestor hits, where the ancestor of the requested
 * item is actually cached.
 * @return {number} The cache's current hit rate.
 */
static getCacheHitRate()

/**
 * Resets the cache's hit rate counters back to zero.
 */
static resetCacheHitRate()

/**
 * Interpolates variables into a template string based on the object's scope (passed into the
 * constructor, if any) and the optional scope argument.  Characters forbidden by Firebase are
 * escaped into a "\xx" hex format.
 *
 * @param  {string} string The template string to interpolate.  You can refer to scope variables
 *     using both ":varName" and "{varName}" notations, with the latter also accepting dot-separated
 *     child attributes like "{varName.child1.child2}".
 * @param  {Object} scope Optional bindings to add to the ones carried by this NodeFire object.
 *     This scope takes precedence if a key is present in both.
 * @return {string} The interpolated string
 */
interpolate(string, scope)

/**
 * Authenticates with Firebase, using either a secret or a custom token.
 * @param  {string} secret A secret for the Firebase referenced by this NodeFire object (copy from
 *     the Firebase dashboard).
 * @param  {Object} authObject Optional.  If provided, instead of authenticating with the secret
 *     directly (which disables all security checks), we'll generate a custom token with the auth
 *     value provided here and an expiry far in the future.
 * @return {Promise} A promise that is resolved when the authentication has completed successfully,
 *     and rejected with an error if it failed.
 */
auth(secret, authObject)

/**
 * Unauthenticates from Firebase.
 * @return {Promise} A resolved promise (for consistency, since unauthentication is immediate).
 */
unauth()

/**
 * Creates a new NodeFire object on the same reference, but with an extended interpolation scope.
 * @param  {Object} scope A dictionary of interpolation variables that will be added to (and take
 *     precedence over) the one carried by this NodeFire object.
 * @return {NodeFire} A new NodeFire object with the same reference and new scope.
 */
scope(scope)

/**
 * Creates a new NodeFire object on a child of this one, and optionally an augmented scope.
 * @param  {string} path The path to the desired child, relative to this reference.  The path will
 *     be interpolated using this object's scope and the additional scope provided.  For the syntax
 *     see the interpolate() method.
 * @param  {Object} scope Optional additional scope that will add to (and override) this object's
 *     scope.
 * @return {NodeFire} A new NodeFire object on the child reference, and with the augmented scope.
 */
child(path, scope)

/**
 * Gets this reference's current value from Firebase, and inserts it into the cache if a
 * maxCacheSize was set and the `cache` option is not false.
 * @return {Promise} A promise that is resolved to the reference's value, or rejected with an error.
 *     The value returned is normalized: arrays are converted to objects, and the value's priority
 *     (if any) is set on a ".priority" attribute if the value is an object.
 */
get()

/**
 * Adds this reference to the cache (if maxCacheSize set) and counts a cache hit or miss.
 */
cache()

/**
 * Removes this reference from the cache (if maxCacheSize is set).
 * @return True if the reference was cached, false otherwise.
 */
uncache()

/**
 * Sets the value at this reference.  To set the priority, include a ".priority" attribute on the
 * value.
 * @param {Object || number || string || boolean} value The value to set.
 * @returns {Promise} A promise that is resolved when the value has been set, or rejected with an
 *     error.
 */
set(value)

/**
 * Sets the priority at this reference.  Useful because you can't pass a ".priority" key to
 * update().
 * @param {string || number} priority The priority for the data at this reference.
 * @returns {Promise} A promise that is resolved when the priority has been set, or rejected with an
 *     error.
 */
setPriority(priority)

/**
 * Updates a value at this reference, setting only the top-level keys supplied and leaving any other
 * ones as-is.
 * @param  {Object} value The value to update the reference with.
 * @return {Promise} A promise that is resolved when the value has been updated, or rejected with an
 *     error.
 */
update(value)

/**
 * Removes this reference from the Firebase.
 * @return {Promise} A promise that is resolved when the value has been removed, or rejected with an
 *     error.
 */
remove()

/**
 * Pushes a value as a new child of this reference, with a new unique key.  Note that if you just
 * want to generate a new unique key you can call generateUniqueKey() directly.
 * @param  {Object || number || string || boolean} value The value to push.
 * @return {Promise} A promise that is resolved to a new NodeFire object that refers to the newly
 *     pushed value (with the same scope as this object), or rejected with an error.
 */
push(value)

/**
 * Runs a transaction at this reference.  The transaction is not applied locally first, since this
 * would be incompatible with a promise's complete-once semantics.
 *
 * There's a bug in the Firebase SDK that fails to update the local value and will cause a
 * transaction to fail repeatedly until it fails with maxretry.  If you specify the detectStuck
 * option then an error with the message 'stuck' will be thrown earlier so that you can try to work
 * around the issue.
 *
 * @param  {function(value):value} updateFunction A function that takes the current value at this
 *     reference and returns the new value to replace it with.  Return undefined to abort the
 *     transaction, and null to remove the reference.  Be prepared for this function to be called
 *     multiple times in case of contention.
 * @param  {Object} options An options objects that may include the following properties:
 *     {number} detectStuck Throw a 'stuck' exception after the update function's input value has
 *         remained unchanged this many times.  Defaults to 0 (turned off).
 *     {boolean} prefetchValue Fetch and keep pinned the value referenced by the transaction while
 *         the transaction is in progress.  Defaults to true.
 *     {number} timeout A number of milliseconds after which to time out the transaction and
 *         reject the promise with 'timeout'.
 * @return {Promise} A promise that is resolved with the (normalized) committed value if the
 *     transaction committed or with undefined if it aborted, or rejected with an error.
 */
transaction(updateFunction, options)

/**
 * Generates a unique string that can be used as a key in Firebase.
 * @return {string} A unique string that satisfies Firebase's key syntax constraints.
 */
generateUniqueKey()

/**
 * Returns the current timestamp after adjusting for the Firebase-computed server time offset.
 * @return {number} The current time in integer milliseconds since the epoch.
 */
now()

/**
 * Returns just the path component of the reference's URL.
 * @return {string} The path component of the Firebase URL wrapped by this NodeFire object.
 */
path()

/* Some methods that work the same as on Firebase objects. */
parent()
root()
toString()
key()
ref()

/* Listener registration methods.  They work the same as on Firebase objects, except that the
   snapshot passed into the callback (and forEach) is wrapped such that:
     1) The val() method will return a normalized method (like NodeFire.get() does).
     2) The ref() method will return a NodeFire reference, with the same scope as the reference
        on which on() was called.
     3) The child() method takes an optional extra scope parameter, just like NodeFire.child().
*/
on(eventType, callback, cancelCallback, context)
off(eventType, callback, context)

/* Query methods, same as on Firebase objects. */
limitToFirst(limit)
limitToLast(limit)
startAt(priority, name)
endAt(priority, name)
equalTo(value, key)
orderByChild(key)
orderByKey()
orderByPriority()
```
