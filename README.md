# Concurrent-Ticket-Booking-System-with-Seat-Locking-and-Confirmation
const express = require('express');
const app = express();

app.use(express.json());

// In-memory seat database
const NUM_SEATS = 10;
const seats = {};
for (let i = 1; i <= NUM_SEATS; i++) {
  seats[i] = { status: 'available', lockExpiresAt: null };
}

// API Endpoints

// View all seats
app.get('/seats', (req, res) => {
  res.json(seats);
});

// Lock a seat
app.post('/lock/:id', (req, res) => {
  const id = req.params.id;
  const seat = seats[id];
  if (!seat) return res.status(400).json({ message: 'Invalid seat id.' });

  // Expire lock if timeout
  if (seat.status === 'locked' && seat.lockExpiresAt && Date.now() > seat.lockExpiresAt) {
    seat.status = 'available';
    seat.lockExpiresAt = null;
  }

  if (seat.status === 'available') {
    seat.status = 'locked';
    seat.lockExpiresAt = Date.now() + 60 * 1000;
    return res.json({ message: `Seat ${id} locked successfully. Confirm within 1 minute.` });
  } else {
    return res.status(400).json({ message: `Seat ${id} is not available.` });
  }
});

// Confirm seat booking
app.post('/confirm/:id', (req, res) => {
  const id = req.params.id;
  const seat = seats[id];
  if (!seat) return res.status(400).json({ message: 'Invalid seat id.' });

  if (seat.status === 'locked' && seat.lockExpiresAt && Date.now() > seat.lockExpiresAt) {
    seat.status = 'available';
    seat.lockExpiresAt = null;
    return res.status(400).json({ message: `Lock expired for seat ${id}. Please lock again.` });
  }

  if (seat.status === 'locked') {
    seat.status = 'booked';
    seat.lockExpiresAt = null;
    return res.json({ message: `Seat ${id} booked successfully!` });
  } else {
    return res.status(400).json({ message: `Seat ${id} is not locked and cannot be booked` });
  }
});

// Serve HTML UI for interacting easily
app.get('/', (req, res) => {
  res.send(`
<!DOCTYPE html>
<html>
<head>
  <title>Seat Booking System</title>
  <script>
    async function viewSeats() {
      const res = await fetch('/seats');
      const seats = await res.json();
      let out = '';
      for (const key in seats) {
        out += \`Seat \${key}: \${seats[key].status}<br>\`;
      }
      document.getElementById('output').innerHTML = out;
    }

    async function lockSeat() {
      const seatId = document.getElementById('seatId').value;
      const res = await fetch('/lock/' + seatId, { method: 'POST' });
      const msg = await res.json();
      document.getElementById('output').innerHTML = msg.message;
    }

    async function confirmSeat() {
      const seatId = document.getElementById('seatId').value;
      const res = await fetch('/confirm/' + seatId, { method: 'POST' });
      const msg = await res.json();
      document.getElementById('output').innerHTML = msg.message;
    }
  </script>
</head>
<body>
  <h1>Seat Booking System</h1>
  <button onclick="viewSeats()">View Seats</button><br><br>
  <input id="seatId" type="number" placeholder="Enter Seat Number (1-10)">
  <button onclick="lockSeat()">Lock Seat</button>
  <button onclick="confirmSeat()">Confirm Seat</button>
  <p id="output"></p>
</body>
</html>
  `);
});

const PORT = 3000;
app.listen(PORT, () => { console.log(\`Server running on port \${PORT}\`); });
