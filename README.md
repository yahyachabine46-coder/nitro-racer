<script src="https://cdn.jsdelivr.net/npm/tesseract.js@5/dist/tesseract.min.js"></script>

<script>
    async function solveScreen() {
        // 1. Capture the screen (The Eyes)
        const stream = await navigator.mediaDevices.getDisplayMedia();
        const track = stream.getVideoTracks()[0];
        const imageCapture = new ImageCapture(track);
        const bitmap = await imageCapture.grabFrame();

        // 2. Read the text (The Recognition)
        const { data: { text } } = await Tesseract.recognize(bitmap);
        console.log("I see this text:", text);

        // 3. Ask the AI (The Brain)
        // You would send 'text' to an AI API here
        const answer = await askAI(text); 
        alert("The Answer is: " + answer);
    }
</script>
