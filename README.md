# watson-speech-streaming-hack
quick and dirty streaming speech capture and relay to chatbot, etc

Do you want to try out streaming audio capture and conversion to text, but not quite ready for building up websockets connections, and handling gnarly API calls?

There is a quick and easy way to get there with IBM Watson Speech to Text, using the very comprehensive [Speech Services for Browsers examples](https://github.com/watson-developer-cloud/speech-javascript-sdk).

To get started, create a free instance of [Watson Speech to Text](https://cloud.ibm.com/catalog/services/speech-to-text) *AND* [Watson Text to Speech](https://cloud.ibm.com/catalog/services/text-to-speech) -- get both, they're free, and it avoids having to modify the exmaple code too much).

Record the services' IAM APIKEY values from the default credentials and be ready to assign them to environment variables (TEXT_TO_SPEECH_IAM_APIKEY and SPEECH_TO_TEXT_IAM_APIKEY) when needed.

Clone the https://github.com/watson-developer-cloud/speech-javascript-sdk repository to your local development system (laptop/desktop).

Follow the setup instructions and try out the live audio conversion straight from your browser to the local application.

Easy, right?

With the default sample pages, all the converted audio output is being sent either to the page, or the browser console; with a minor tweak, you can send the text to a chatbot text endpoint.

# Example modification to speech-enable a chatbot

To get going with minimal fuss, let's make a couple of modifications to the `examples/static/microphone-streaming-object-to-console.html` page. The page name pretty much says what it does --
1. captures streaming audio through the browser [getUserMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia) microphone support
1. stream the audio to Watson Speech to Text service
1. capture response JSON objects with converted text (including "final" indicator suggesting end of sentence)
1. log raw JSON object to browser console

The hack here is to take the JSON object text content and forward this to a chatbot endpoint, providing a simple speech-enablement.
The assumption is that the chatbot offers a raw web-form interface - a GET/POST loop with a text input field called `query`; the text will be sent using `application/x-www-form-urlencoded` Content-Type.

Here's the off-the-shelf code in the example
```
<pre><code><script style="display: block;">
document.querySelector('#button').onclick = function () {

  fetch('/api/speech-to-text/token')
  .then(function(response) {
      return response.json();
  }).then(function (token) {

    var stream = WatsonSpeech.SpeechToText.recognizeMicrophone(Object.assign(token, {
        objectMode: true, // send objects instead of text
        format: false // optional - performs basic formatting on the results such as capitals an periods
    }));

    stream.on('data', function(data) {
      console.log(data);
    });

    stream.on('error', function(err) {
        console.log(err);
    });

    document.querySelector('#stop').onclick = stream.stop.bind(stream);

  }).catch(function(error) {
      console.log(error);
  });
};

</script></code></pre>
```

Modify the `stream.on('data') function to extract the text from Watson STT, and POST to your chatbot input form:
```
  stream.on('data', function(data) {
    console.log(data);
    if (data.results[0].final){
        var query = data.results[0].alternatives[0].transcript;
        console.log(query);
        var xhr = new XMLHttpRequest();
        xhr.open('POST', "https:/<your-chatbot-input-form>/chat-input", true);
        xhr.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');
        xhr.onreadystatechange = function() {
          if (xhr.readyState == XMLHttpRequest.DONE) {
            document.querySelector('#output').innerHTML = this.responseText;
          }
        }
        xhr.send("query="+ query);
      }
 });
```

(This is minimal to demonstrate the concept - modify as you see fit to make use of more reliable overlays to XMLHttpRequest  - jquery, etc).

If your chatbot sends back response text, the `onreadystatechange` callback will catch that, and update the page.
