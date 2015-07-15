# Opener Manifest

[Opener](http://www.opener.link) is an app for iOS that allows people to open web links in native apps instead. It does so by transforming web links within an engine powered by a rule set. This repo is the public version of that rule set.

## Apps

The `apps` top level key in the manifest contains an ordered list of dictionaries, each representing an app supported by Opener. Each app contains the following fields

- `identifier` *string*: A human-readable identifier for this app, used elsewhere in the manifest.
- `name` *string*: The user-facing name for this app within Opener.
- `storeIdentifier` *number as string*: The identifier of the app on the App Store.
- `scheme` *URL as string*: A URL containing only the scheme that will open this app.
- `new` *bool*: Indicates whether or not this app will be include in the "New Apps" group in Opener. Evaluates to `false` if unspecified.
- `platform` *string*: Specifies if this app should only show up on iPhone/iPod Touch (value=`phone`) or on iPad (value=`pad`), shows on both if unspecified. *(Opener 1.0.1 and above)*

For example, if Opener were to include itself as an app

```
{
	"identifier": "opener",
	"storeIdentifier": "989565871",
	"name": "Opener",
	"scheme": "opener://",
	"new": true
}
```


## Actions

The `actions` top level key in the manifest contains a list of dictionaries, each corresponding to a web URL-to-native URL rule. There's a many-to-one relationship between the values in `actions` and `apps`.

### Common values

- `title` *string*: The user-facing title for this action.
- `regex` *string*: A regular expression string that the input URL is matched against. If this regex is matched by Opener for a given input, this action will appear in the list of available opening options.
- `includeHeaders` *bool*: Indicates if headers should be included in the string that `regex` is matched with. If `true`, the headers are included in the input as a JSON encoded string separated from the input URL by a newline. *(Opener 1.0.2 and above)*
- `formats` *array of dictionaries*: Specifies the apps that an action can be opened in (see [below](#formats)).

### <a tag="formats">Formats</a>

Because an action could taken in multiple apps, there's an array within each action dictionary named `formats`. Each entry in this array matches the input URL with an app-specific output for the given action. Each of these contains the following keys.

- `appIdentifier` *string*: The identifier of the app that this action applies to. Should match the `identifier` of an app.
- `format` *string*: The regex template applied to the input. Mutually exclusive with `script`.

### Advanced URL generation in formats

Some app native URLs can't be generated using simple regex templating, they require lookups or encoding of some sort. To do this, action formats can provide Javascript methods that are executed to convert input URLs to app native action URLs.

- `script` *Javascript string*: Mutually exclusive with `format`.

This script must contain a Javascript function named `process` that takes two inputs, a URL and an anonymous function to be called upon completion. Once complete, the completion handler should be called passing the result or `null` on failure.

For example

```
function process(url, completionHandler) {
	// do something with URL...
	url = rot13(url);
	
	completionHandler(url);
}
```

Opener enforces a timeout of 15 seconds if `completionHandler` isn't called.

#### Common scenarios

If you're planning on using the **Twitter Cards** tags embedded on a page, you should isolate the app's URL scheme and do something like this, which is used for [Swarm](https://www.swarmapp.com/).

```
function process(url, completionHandler) { var xmlhttp = new XMLHttpRequest(); xmlhttp.onreadystatechange = function() { if (xmlhttp.readyState == 4 && xmlhttp.status == 200) { var res = xmlhttp.responseText; var regex = RegExp('.*(swarm:\/\/.*?)\".*'); var match = regex.exec(res)[1]; completionHandler(match); } }; xmlhttp.open('GET', url, true); xmlhttp.send(); }
```

If you need to **URL encode** something, here's an example for [Overcast](https://overcast.fm/).

```
function process(url, completionHandler) { completionHandler('overcast://x-callback-url/add?url=' + encodeURIComponent(url)); }
```

If you need to pick a component out of the URL passed in, you can use the Javascript `RegExp` class.

### Testing

To keep Opener maintainable, tests for actions can and should be provided.

At the `action` level:

- `testInputs` *array of strings*: An array of test inputs that will be run against `regex` then each action.

At the `format` level:

- `testResults` *array of strings or null entries*:  An array of expected results for this format for each of the test inputs. `null` should be used to specify that a test input *should not* match.

For example

```
{
	...
	"regex": "http(?:s)?://(?:www\\.)?foo\.bar/(\\d+).*$",
	"testInputs": [
		"https://foo.bar/1234"
		"http://www.foo.bar/wat"
	],
	"formats": [
		{
			...
			"format": "foo-app://entry/$1",
			"testResults": [
				"foo-app://entry/1234",
				null
			]
		},
		{
			...
			"script": "function process(url, completion) { completion('bar-app://' + encodeURIComponent(url)); }",
			"testResults": [
				"bar-app://https%3A%2F%2Ffoo.bar%2F1234",
				null
			]
		}
	]
}
```


Testing formats that have `includeHeaders` is not currently possible.

## Contributing

Pull requests are welcome! Because Opener is a closed source app with an experience that I'd like to keep great, I'm going to be pedantic about these requests. I will likely manipulate the order of the apps and actions that are added, and handle the `new` flag for them.