#Xively Energia library

A library for Energia to make it easier to talk to Xively.

_This library requires [HTTP Client](https://github.com/zlalanne/HttpClient)._  
_This library is a fork of [xively_arduino](https://github.com/xively/xively_arduino)._

##Features

   1. Generic functions for:
   	- Uploading datapoints
   	- Downloading datapoints
   2. Compatible with:
    - MSP430F5529 LaunchPad
    - CC3000 BoosterPack

##For a Quickstart Example
Look no further!  If you want a quick example, connect your LaunchPad to your computer and try out one of the examples included with this library.

>In Energia, go to Files > Examples and choose WiFiDatastreamDownload or WiFiDatastreamUpload from the xively_energia library folder

##Setup Your Sketch

**1. Specify your API key and Feed ID**
```c
char xivelyKey[] = "YOUR_XIVELY_API_KEY";
// Should be something like "HsNiCoe_Es2YYWltKeRFPZL2xhqSAKxIV21aV3lTL2h5OD0g"
#define FEED_ID XXXXXX
// The 3 to 6-digit number (like 504 or 104097), that identifies the Xively Feed you're using
```

**2. Create IDs for your datastreams as `char` arrays (or `String` objects for a `String` datastream)**

In Xively, the name of a datastream is known as the **Stream ID**.  In the example below, we'll give the datastreams names by setting their **Stream IDs** as "humidity", "temperature", "my_thoughts_on_the_temperature" and "more_thoughts".

```c
// For datastreams of floats:
char myFloatStream[] = "humidity";
// For datastreams of ints:
char myIntStream[] = "temperature";
// For datastreams of Strings:
String myStringStream("my_thoughts_on_the_temperature");
// For datastreams of char buffers:
char myCharBufferStream[] = "more_thoughts";  // ID of the array
const int bufferSize = 140;                   // size of the array
char bufferValue[bufferSize];                 // the array of chars itself
```

`String` datastreams and `char` buffer datastreams are similar: both will be able to send strings to Xively datastreams. For beginners, using `String` datastreams will be fine much of the time.

Using char buffers reduces the memory footprint of your sketch by not requiring the String library.  Also, using char buffers allows you to specify exactly how much memory is used for a datapoint, so you don't accidentally overflow the MSP430's mem capacity with a huge string datapoint.  It's a little bit harder to understand for beginners -- consult XivelyDatastream.cpp for info.

**3. Create an array of `XivelyDatastream` objects**

```c
XivelyDatastream datastreams[] = {
  // Float datastreams are set up like this:
  XivelyDatastream(myFloatStream, strlen(myFloatStream), DATASTREAM_FLOAT),
  // Int datastreams are set up like this:
  XivelyDatastream(myIntStream, strlen(myIntStream), DATASTREAM_INT),
  // String datastreams are set up like this:
  XivelyDatastream(myStringStream, DATASTREAM_STRING),
  // Char buffer datastreams are set up like this:
  XivelyDatastream(myCharBufferStream, strlen(myCharBufferStream), DATASTREAM_BUFFER, bufferValue, bufferSize),
};
```

`XivelyDatastream` objects can contains some or all of the following variables, depending on what type of datapoints are in the datastream (see above example for which are required):

| | Variable | Type | Description |
|---|---|:---:|---|
| 1     | aIdBuffer | char*|`char` array containing the ID of the datastream
| 2     | aIdBufferLength |  int |for `int` or `float` datastreams only: the number of  `char` in the datastream's ID
| 3 | aType | int |**0** or DATASTREAM_STRING for a String; **1** or DATASTREAM_BUFFER for a char buffer; **2** or DATASTREAM_INT for an int; **3** or DATASTREAM_FLOAT for a float
| 4 | aValueBuffer | char* | A `char` array, _aValueBufferLength_ elements long
| 5 | aValueBufferLength | int | The number of elements in the `char` array


**4. Last, wrap this array of `XivelyDatastream` objects into a `XivelyFeed`**

Unlike the **Stream ID**, which is what a _Datastream's_ name is stored as, the **ID** of a _Feed_ is a number which is used to uniquely identify which Xively Feed you are addressing.  For  example, a Feed **ID** of 504 would mean that you were using the feed at xively.com/feeds/504.

```c
XivelyFeed feed(FEED_ID, datastreams, 4);
```

| | Variable | Type | Description |
|---|---|:---:|---|
| 1     | aID | unsigned long | The Feed's **ID**, as defined at the top of your sketch
| 2     | aDatastreams | XivelyDatastream* |Your `XivelyDatastream` array
| 3 | aDatastreamsCount | int | How many datastreams are in the array

**5. Instantiate the library's Xively client**

Be sure to enter your SSID and password as the library requires, and then:
>If you're using the built-in WiFi library:
```c
WiFiClient client;
XivelyClient xivelyclient(client);
```

##Sending and Retrieving Xively Datapoints

###Read a Datapoint
```c
Serial.print("My return is: ");

float float_value = datastreams[0].getFloat();        // Retrieve the latest datapoint in a float datastream
int int_value = datastreams[1].getInt();              // Retrieve the latest datapoint in an int datastream
String string_value = datastreams[2].getString();     // Retrieve the latest datapoint in a String datastream
char[140] char_buffer = datastreams[3].getBuffer();   // Retrieve the latest datapoint in a char buffer datastream
```

###Update a Datapoint

The library makes it easy to upload data as strings or numbers.
```c
datastreams[0].setFloat(1.5);                           // Push a float datapoint
datastreams[1].setInt(23);                              // Push an int datapoint
datastreams[2].setString("Pretty comfy temperature");   // Push a String datapoint
datastreams[3].setBuffer("But quite dry");              // Push a char buffer datapoint
```

##Error codes

This library uses [_zlalanne/HttpClient_](https://github.com/zlalanne/HttpClient/blob/master/HttpClient.h) which has following error codes:

* `HTTP_SUCCESS = 0` - no error
* `HTTP_ERROR_CONNECTION_FAILED = -1` - connection to _api.xively.com_ has failed
* `HTTP_ERROR_API = -2` - a method of _HttpClient_ class was called incorrectly
* `HTTP_ERROR_TIMED_OUT = -3` - connection with _api.xively.com_ has timed-out
* `HTTP_ERROR_INVALID_RESPONSE = -4` - invalid or unexpected response from the server

Apart from the above, the library will convert any _non-2xx_ status code to a nagative value like so:
```c
ret = http.responseStatusCode();
if ((ret < 200) || (ret > 299))
{

  if (ret > 0)
  {
    ret = ret * -1;
  }
}
```

Therefore:
* if we got a _2xx_, we will return that as is
* if we got a negative value (e.g. `HTTP_ERROR_CONNECTION_FAILED`, `HTTP_ERROR_TIMED_OUT` or other), we will return that as is
* any _non-2xx_ status code is returned multiplied by _-1_, i.e. a _403 (Authentication error)_ will be returned as _-403_

[![githalytics.com alpha](https://cruel-carlota.pagodabox.com/50cb84588d3ed6bc47fd176ad319bb20 "githalytics.com")](http://githalytics.com/zlalanne/xively_energia.git)
