<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <title>Classroom Queue</title>
  <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-firestore-compat.js"></script>
  <style>
    body { font-family: sans-serif; max-width: 600px; margin: auto; padding: 20px; }
    .entry { padding: 10px; border-bottom: 1px solid #ccc; display: flex; justify-content: space-between; }
    button { margin-left: 10px; }
  </style>
</head>
<body>
  <h1>Classroom Queue</h1>

  <h2>Join Queue</h2>
  <input id="name" placeholder="Your name" /><br><br>
  <input id="comment" placeholder="What do you need help with?" /><br><br>
  <button onclick="joinQueue()">Join</button>

  <h2>Your Position</h2>
  <div id="position">Not in queue</div>

  <h2>Teacher View</h2>
  <div id="queue"></div>

  <script>
    // TODO: replace with your Firebase config
    const firebaseConfig = {
      apiKey: "YOUR_KEY",
      authDomain: "YOUR_DOMAIN",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_BUCKET",
      messagingSenderId: "YOUR_SENDER_ID",
      appId: "YOUR_APP_ID"
    };

    firebase.initializeApp(firebaseConfig);
    const db = firebase.firestore();

    let myId = null;

    async function joinQueue() {
      const name = document.getElementById('name').value;
      const comment = document.getElementById('comment').value;

      const doc = await db.collection('queue').add({
        name,
        comment,
        timestamp: Date.now()
      });

      myId = doc.id;
    }

    db.collection('queue').orderBy('timestamp').onSnapshot(snapshot => {
      const queue = [];
      snapshot.forEach(doc => queue.push({ id: doc.id, ...doc.data() }));

      // teacher view
      document.getElementById('queue').innerHTML = queue.map(e => `
        <div class='entry'>
          <div><strong>${e.name}</strong>: ${e.comment}</div>
          <button onclick="helped('${e.id}')">Helped</button>
        </div>
      `).join('');

      // student view
      if (myId) {
        const index = queue.findIndex(e => e.id === myId);
        if (index === -1) {
          document.getElementById('position').innerText = 'You have been helped!';
        } else {
          document.getElementById('position').innerText = `${index} people ahead of you`;
        }
      }
    });

    function helped(id) {
      db.collection('queue').doc(id).delete();
    }
  </script>
</body>
</html>
