const recordAudio = () => {
    return new Promise(async resolve => {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        const mediaRecorder = new MediaRecorder(stream);
        const audioChunks = [];

        mediaRecorder.addEventListener("dataavailable", event => {
            audioChunks.push(event.data);
        });

        const start = () => mediaRecorder.start();

        const stop = () => {
            return new Promise(resolve => {
                mediaRecorder.addEventListener("stop", () => {
                    const audioBlob = new Blob(audioChunks);
                    const audioUrl = URL.createObjectURL(audioBlob);
                    const audio = new Audio(audioUrl);
                    const reader = new FileReader();
                    reader.readAsDataURL(audioBlob);
                    reader.onloadend = () => {
                        const base64data = reader.result.split(',')[1];
                        resolve({ audioBlob, audioUrl, audio, base64data });
                    };
                });

                mediaRecorder.stop();
            });
        };

        resolve({ start, stop });
    });
};

const recognizeSpeech = async (base64data) => {
    const response = await fetch('https://speech.googleapis.com/v1/speech:recognize?key=YOUR_API_KEY', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            config: {
                encoding: 'LINEAR16',
                sampleRateHertz: 16000,
                languageCode: 'en-US'
            },
            audio: {
                content: base64data
            }
        })
    });

    const data = await response.json();
    return data.results[0].alternatives[0].transcript;
};

const main = async () => {
    const recorder = await recordAudio();
    console.log("Please say something:");
    recorder.start();

    setTimeout(async () => {
        const audio = await recorder.stop();
        console.log("Recognizing...");
        const transcript = await recognizeSpeech(audio.base64data);
        console.log("You said: " + transcript);
    }, 5000);
};

main();
