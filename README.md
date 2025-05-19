
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Object Classifier with Voice Output</title>
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;700&display=swap');
    * {
      box-sizing: border-box;
    }

    body {
      margin: 0;
      font-family: 'Poppins', sans-serif;
      background: linear-gradient(135deg, #3a1c71, #d76d77, #ffaf7b);
      color: white;
      min-height: 100vh;
      display: flex;
      flex-direction: column;
      align-items: center;
      padding: 1rem;
    }

    h1 {
      margin-bottom: 0.5rem;
      font-weight: 700;
      text-shadow: 1px 1px 4px rgba(0,0,0,0.6);
    }

    #camera-container {
      position: relative;
      margin: 1rem 0;
      border-radius: 12px;
      overflow: hidden;
      box-shadow: 0 8px 20px rgba(0,0,0,0.3);
      width: 320px;
      height: 240px;
      background: black;
    }

    video, canvas {
      width: 320px;
      height: 240px;
      border-radius: 12px;
      object-fit: cover;
    }

    #buttons {
      margin: 1rem 0;
      display: flex;
      gap: 1rem;
      flex-wrap: wrap;
      justify-content: center;
    }

    button {
      background-color: #ffaf7b;
      border: none;
      padding: 0.6rem 1.2rem;
      border-radius: 30px;
      color: #3a1c71;
      font-weight: 600;
      font-size: 1rem;
      cursor: pointer;
      box-shadow: 0 5px 8px rgba(0,0,0,0.2);
      transition: background-color 0.3s ease;
    }

    button:hover {
      background-color: #d76d77;
      color: white;
    }

    #result {
      margin-top: 1rem;
      background-color: rgba(255, 255, 255, 0.15);
      padding: 1rem 1.5rem;
      border-radius: 12px;
      font-size: 1.25rem;
      min-height: 48px;
      box-shadow: inset 0 0 10px rgba(0,0,0,0.15);
      text-align: center;
      user-select: none;
    }

    input[type="file"] {
      display: none;
    }

    label[for="upload"] {
      background-color: #d76d77;
      padding: 0.6rem 1.2rem;
      border-radius: 30px;
      font-weight: 600;
      cursor: pointer;
      color: white;
      box-shadow: 0 5px 8px rgba(0,0,0,0.2);
      user-select: none;
      transition: background-color 0.3s ease;
    }

    label[for="upload"]:hover {
      background-color: #ffaf7b;
      color: #3a1c71;
    }

    #loading {
      margin-top: 1rem;
      font-size: 1rem;
      font-style: italic;
      opacity: 0.8;
    }
  </style>
</head>
<body>
  <h1>Object Classifier with Voice Output</h1>
  <div id="camera-container">
    <video id="video" autoplay playsinline muted></video>
    <canvas id="canvas" style="display:none;"></canvas>
  </div>

  <div id="buttons">
    <button id="capture">Capture Photo</button>
    <label for="upload">Upload Photo</label>
    <input type="file" id="upload" accept="image/*" />
    <button id="classify" disabled>Classify Image</button>
    <button id="speak" disabled>Speak Result</button>
  </div>

  <div id="result">Take or upload a photo to classify</div>
  <div id="loading"></div>

  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.8.0/dist/tf.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/mobilenet@2.1.0/dist/mobilenet.min.js"></script>
  <script>
    const video = document.getElementById('video');
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    const captureBtn = document.getElementById('capture');
    const classifyBtn = document.getElementById('classify');
    const speakBtn = document.getElementById('speak');
    const uploadInput = document.getElementById('upload');
    const resultDiv = document.getElementById('result');
    const loadingDiv = document.getElementById('loading');

    let model;
    let currentImage; // Image object with captured or uploaded image
    let classificationResult = null;

    async function setupModel() {
      loadingDiv.textContent = 'Loading MobileNet model, please wait...';
      model = await mobilenet.load();
      loadingDiv.textContent = '';
    }

    async function startCamera() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' }, audio: false });
        video.srcObject = stream;
        await video.play();
      } catch (error) {
        loadingDiv.textContent = 'Could not access camera: ' + error.message;
      }
    }

    // Capture the current video frame into canvas and update currentImage
    function capturePhoto() {
      if (!video.srcObject) {
        resultDiv.textContent = 'Camera not available';
        return;
      }
      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      // Create an Image object from canvas data
      currentImage = new Image();
      currentImage.src = canvas.toDataURL('image/png');
      currentImage.onload = () => {
        classifyBtn.disabled = false;
        speakBtn.disabled = true;
        resultDiv.textContent = 'Photo captured. Ready to classify.';
        classificationResult = null;
      };
    }

    // When user uploads a photo
    uploadInput.addEventListener('change', (e) => {
      const file = e.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = function(event) {
          currentImage = new Image();
          currentImage.src = event.target.result;
          currentImage.onload = () => {
            // Draw uploaded image on canvas for consistent processing
            canvas.width = currentImage.width;
            canvas.height = currentImage.height;
            ctx.clearRect(0,0,canvas.width,canvas.height);
            ctx.drawImage(currentImage, 0, 0, canvas.width, canvas.height);
            classifyBtn.disabled = false;
            speakBtn.disabled = true;
            resultDiv.textContent = 'Photo uploaded. Ready to classify.';
            classificationResult = null;
          };
        };
        reader.readAsDataURL(file);
      }
    });

    async function classifyImage() {
      if (!model) {
        resultDiv.textContent = 'Model not loaded yet.';
        return;
      }
      if (!currentImage) {
        resultDiv.textContent = 'No image to classify.';
        return;
      }
      loadingDiv.textContent = 'Classifying image...';
      classifyBtn.disabled = true;
      // Use tf.browser.fromPixels to create tensor from image in canvas
      const imgTensor = tf.browser.fromPixels(canvas);
      try {
        const predictions = await model.classify(imgTensor);
        if (predictions.length > 0) {
          classificationResult = predictions[0].className;
          resultDiv.textContent = 'Result: ${classificationResult}'
          speakBtn.disabled = false;
        } else {
          resultDiv.textContent = 'Could not classify the image.';
          speakBtn.disabled = true;
          classificationResult = null;
        }
      } catch (err) {
        resultDiv.textContent = 'Error during classification: ' + err.message;
        classificationResult = null;
        speakBtn.disabled = true;
      } finally {
        loadingDiv.textContent = '';
        classifyBtn.disabled = false;
        imgTensor.dispose();
      }
    }

    function speakResult() {
      if (!classificationResult) return;
      const utterance = new SpeechSynthesisUtterance(classificationResult);
      utterance.lang = 'id-ID'; // Indonesian locale
      speechSynthesis.speak(utterance);
    }


    // Event listeners
    captureBtn.addEventListener('click', capturePhoto);
    classifyBtn.addEventListener('click', classifyImage);
    speakBtn.addEventListener('click', speakResult);

    // Initialize
    setupModel();
    startCamera();
  </script>
</body>
</html>
