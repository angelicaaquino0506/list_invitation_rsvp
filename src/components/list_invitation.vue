<template>
    <div class="rsvp-container celebration-gradient">
      <h2 class="title-reveal">Angela's Enchanted 18th Celebration</h2>
  
      <div class="attendee-summary card-grid">
        <div class="summary-card pop-bounce">
          <i class="fas fa-users-crown icon-large"></i>
          <span>Total Attendees: {{ totalAttendeesCount }}</span>
        </div>
        <div class="summary-card pop-bounce">
          <i class="fas fa-sparkles icon-large"></i>
          <span>Candles Confirmed: {{ eighteenCandlesAttendedCount }}</span>
        </div>
        <div class="summary-card pop-bounce">
          <i class="fas fa-calendar-check icon-large"></i>
          <span>Event Status: Live!</span>
        </div>
      </div>
  
      <div class="attendee-lists">
        <h3 class="slide-in-heading">The Royal Guest Scroll</h3>
  
        <div class="category pulse-border">
          <h4><i class="fas fa-star-shooting category-icon"></i> The 18 Dazzling Candles</h4>
          <div v-if="eighteenCandlesWithAttendance.length" class="attendee-table-container">
            <table class="attendee-table shimmer-effect">
              <thead>
                <tr>
                  <th>Name</th>
                  <th style="text-align: center;" class="text-center">Attended?</th>
                  <!-- <th style="text-align: center;" class="text-center">Confirmed At</th> -->
                </tr>
              </thead>
              <tbody>
                <tr v-for="person in eighteenCandlesWithAttendance" :key="person.id" class="list-item-fade">
                  <td>{{ person.name }}</td>
                  <td class="text-center">
                    <!-- <span v-if="person.attended" class="attendance-tag tag-attended swing-in">Yes!</span>
                    <span v-else class="attendance-tag tag-pending swing-in">Pending</span> -->
                    <span lass="attendance-tag tag-attended swing-in">{{ person.attended ? person.attended : '' }}</span>
                  </td>
                  <!-- <td class="text-center">{{ person.timeAccepted }}</td> -->
                </tr>
              </tbody>
            </table>
          </div>
          <p v-else class="empty-message fade-in-text">The 18 Dazzling Candles await their grand entry!</p>
        </div>
  
        <div class="category pulse-border">
          <h4><i class="fas fa-heart category-icon"></i> The 8 Beloved Candles</h4>
          <ul v-if="eightCandles.length">
            <li v-for="person in eightCandles" :key="person.id" class="list-item-fade">
              <span>{{ person.name }}</span>
              <button @click="removeAttendee(person.id)" class="remove-btn wiggle-on-hover">
                <i class="fas fa-minus-circle"></i>
              </button>
            </li>
          </ul>
          <p v-else class="empty-message fade-in-text">The 8 Beloved Candles are yet to grace us with their presence.</p>
        </div>
  
        <div class="category pulse-border">
          <h4><i class="fas fa-champagne-glasses category-icon"></i> Esteemed Guests</h4>
          <ul v-if="guests.length">
            <li v-for="person in guests" :key="person.id" class="list-item-fade">
              <span>{{ person.name }}</span>
              <button @click="removeAttendee(person.id)" class="remove-btn wiggle-on-hover">
                <i class="fas fa-minus-circle"></i>
              </button>
            </li>
          </ul>
          <p v-else class="empty-message fade-in-text">Our esteemed guests are still sending their RSVPs!</p>
        </div>
      </div>
    </div>
  </template>
  
  
  <script setup lang="ts">
  import { ref, computed, onMounted, onUnmounted } from 'vue';
  import { collection, onSnapshot } from 'firebase/firestore';
  import { db } from '@/firebase';
  
  // 1. Define Interfaces for Type Safety
  interface Attendee {
    id: string; // Changed to string to match Firestore doc.id
    name: string;
    role: 'Guest' | 'One of the 8 Candles';
  }
  
  interface EighteenCandleAttendee {
    id: string; // Changed to string to match Firestore doc.id
    name: string;
    attended: boolean;
    notAttended: boolean;
    timeAccepted: string;
  }
  
  // 2. Reactive State
  const usersList = ref<any[]>([]);
  const attendees = ref<Attendee[]>([]); // For 'One of the 8 Candles' and 'Guests'
  const eighteenCandlesWithAttendance = ref<EighteenCandleAttendee[]>([]);
  
  // Predefined 18 Candles data
  const eighteenRosesData = ref<{ name: string; value: string }[]>([
    { name: 'Mr. Arjay magno', value: 'Arjay magno' },
    { name: 'Mr. Reynaldo Aquino', value: 'Reynaldo Aquino' },
    { name: 'Mr. Renz Gonzales', value: 'Renz Gonzales' },
    { name: 'Mr. Ronald Villio', value: 'Ronald Villio' },
    { name: 'Mr. Efren Rodriguez', value: 'Efren Rodriguez' },
    { name: 'Mr. Ralph Louren Cincua', value: 'Ralph Louren Cincua' },
    { name: 'Mr. Wingard Baraquiel', value: 'Wingard Baraquiel' },
    { name: 'Mr. Rexie Van Galanta', value: 'Rexie Van Galanta' },
    { name: 'Mr. John Lumbis', value: 'John Lumbis' },
    { name: 'Mr. Aaron Josh Corbita', value: 'Aaron Josh Corbita' },
    { name: 'Mr. Marvy James Bayani', value: 'Marvy James Bayani' },
    { name: 'Mr. Jhon Ryan Dellosmas', value: 'Jhon Ryan Dellosmas' },
    { name: 'Mr. Wilfredo Andres', value: 'Wilfredo Andres' },
    { name: 'Mr. Jasper Panganiban', value: 'Jasper Panganiban' },
    { name: 'Mr. Prince Charles Giron', value: 'Prince Charles Giron' },
    { name: 'Ms. Ma. Zarah Muli', value: 'Zarah Muli' },
    { name: 'Mrs. Maria Thelma Muli', value: 'Maria Thelma Muli' },
    { name: 'Mr. Angelo Muli', value: 'Angelo Muli' },
  ]);
  
  let unsubscribe: (() => void) | null = null; // To store the unsubscribe function
  
  // Function to set up the real-time listener for users
  const setupUsersListener = () => {
    const usersCollectionRef = collection(db, "users");
    unsubscribe = onSnapshot(usersCollectionRef, (querySnapshot) => {
      const fetchedUsers: any[] = [];
      querySnapshot.forEach((doc) => {
        fetchedUsers.push({ id: doc.id, ...doc.data() });
      });
      usersList.value = fetchedUsers;
      console.log("Real-time fetched users:", usersList.value);
  
      // Re-process 18 Candles data whenever usersList changes
      processEighteenCandles();
    }, (error) => {
      console.error("Error fetching real-time RSVPs: ", error);
    });
  };
  
  // Function to process 18 Candles data with fetched attendance
  const processEighteenCandles = () => {
    eighteenCandlesWithAttendance.value = eighteenRosesData.value.map(rose => {
      const matchedUser = usersList.value.find(user =>
        user.name && rose.name.toLowerCase().includes(user.name.toLowerCase())
      );
  
      return {
        id: matchedUser ? matchedUser.id : rose.value, // Use Firestore ID if matched, otherwise a fallback
        name: rose.name,
        attended: matchedUser ? matchedUser.attended : false,
        notAttended: matchedUser ? !matchedUser.attended : false,
        timeAccepted: matchedUser && matchedUser.attended && matchedUser.timeAccepted
          ? new Date(matchedUser.timeAccepted).toLocaleString('en-US', {
              hour: '2-digit',
              minute: '2-digit',
              second: '2-digit',
              hour12: true,
            })
          : '',
      };
    });
  };
  
  // 3. Computed Properties for Categorization (remain largely the same)
  const eightCandles = computed(() =>
    attendees.value.filter(a => a.role === 'One of the 8 Candles')
  );
  
  const guests = computed(() =>
    attendees.value.filter(a => a.role === 'Guest')
  );
  
  // 4. Methods (addAttendee and removeAttendee for general attendees can remain as is if you're not saving them to Firestore)
  // If you want to save 'One of the 8 Candles' and 'Guests' to Firestore, you'll need to implement those functions separately.
  
  const removeAttendee = (idToRemove: string) => {
    attendees.value = attendees.value.filter(attendee => attendee.id !== idToRemove);
    // If these are also in Firestore, you'd add Firestore delete logic here.
  };

  const totalAttendeesCount = computed(() => {
    // Assuming all listed 18 candles are "attendees" for total count purposes,
    // plus the 8 candles and guests from Firestore.
    return eighteenCandlesWithAttendance.value.length + eightCandles.value.length + guests.value.length;
  });
  
  const eighteenCandlesAttendedCount = computed(() => {
    return eighteenCandlesWithAttendance.value.filter(p => p.attended).length;
  });
  
  // Lifecycle hooks
  onMounted(() => {
    setupUsersListener(); // Start listening for real-time updates when the component is mounted
  });
  
  onUnmounted(() => {
    if (unsubscribe) {
      unsubscribe(); // Clean up the listener when the component is unmounted
      console.log("Firestore listener unsubscribed.");
    }
  });
  </script>
  
  <style scoped>
  /* Import Google Fonts for a playful, modern feel */
  @import url('https://fonts.googleapis.com/css2?family=Pacifico&family=Quicksand:wght@300;400;600;700&display=swap');
  
  /* Add Font Awesome for icons */
  @import url('https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.3/css/all.min.css');
  
  :root {
    --primary-fuchsia: #e7008a; /* Vibrant pink/fuchsia */
    --secondary-deep-blue: #4a00e0; /* Deep, enchanting blue */
    --accent-gold: #FFD700; /* Bright gold for highlights */
    --accent-light: #fdf5e6; /* Light cream/beige for softer elements */
    --text-dark: #333;
    --text-light: #fff;
    --bg-card: rgba(255, 255, 255, 0.95); /* Slightly transparent white for cards */
    --border-card: rgba(231, 0, 138, 0.3); /* Transparent fuchsia border */
  }
  
  body {
    font-family: 'Quicksand', sans-serif;
    color: var(--text-dark);
    margin: 0;
    padding: 0;
    overflow-x: hidden; /* Prevent horizontal scroll from animations */
  }
  
  /* --- Amazing Background (Gradient + Subtle Pattern) --- */
  .celebration-gradient {
    background: linear-gradient(135deg, var(--primary-fuchsia) 0%, var(--secondary-deep-blue) 100%);
    position: relative;
    min-height: 100vh;
    padding: 50px 20px;
    overflow: hidden; /* Keep particles within */
  }
  
  /* Add a subtle, animated particle effect */
  .celebration-gradient::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-image: radial-gradient(circle, rgba(255, 255, 255, 0.1) 1px, transparent 1px);
    background-size: 20px 20px;
    animation: backgroundPan 60s linear infinite;
    opacity: 0.7;
  }
  
  @keyframes backgroundPan {
    from { background-position: 0 0; }
    to { background-position: 600px 600px; } /* Adjust values for speed/direction */
  }
  
  /* --- Main Container --- */
  .rsvp-container {
    max-width: 1000px;
    margin: 0 auto;
    padding: 30px;
    border-radius: 20px;
    backdrop-filter: blur(10px); /* Frosted glass effect */
    background-color: rgba(255, 255, 255, 0.1); /* Very subtle transparent background */
    box-shadow: 0 20px 50px rgba(0, 0, 0, 0.3); /* Deeper shadow */
    border: 2px solid rgba(255, 255, 255, 0.3); /* Lighter border */
  }
  
  /* --- Heading --- */
  h2 {
    font-family: 'Pacifico', cursive;
    color: var(--accent-light);
    margin-bottom: 40px;
    text-align: center;
    font-size: 3.8rem;
    font-weight: 400;
    letter-spacing: 2px;
    text-shadow: 4px 4px 10px rgba(0, 0, 0, 0.4);
    position: relative;
  }
  
  /* Title Reveal Animation */
  .title-reveal {
    opacity: 0;
    transform: translateY(-50px) scale(0.8);
    animation: titleReveal 1.2s cubic-bezier(0.68, -0.55, 0.265, 1.55) forwards;
  }
  
  @keyframes titleReveal {
    0% { opacity: 0; transform: translateY(-50px) scale(0.8); }
    70% { opacity: 1; transform: translateY(10px) scale(1.05); }
    100% { opacity: 1; transform: translateY(0) scale(1); }
  }
  
  /* --- Summary Cards Grid --- */
  .card-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 25px;
    margin-bottom: 50px;
  }
  
  .summary-card {
    background: var(--bg-card);
    padding: 25px;
    border-radius: 15px;
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15);
    display: flex;
    flex-direction: column;
    align-items: center;
    color: var(--secondary-deep-blue);
    font-size: 1.2rem;
    font-weight: 600;
    border: 1px solid var(--border-card);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
  }
  
  .summary-card:hover {
    transform: translateY(-8px) scale(1.02);
    box-shadow: 0 12px 25px rgba(0, 0, 0, 0.25);
  }
  
  .icon-large {
    font-size: 3rem;
    margin-bottom: 15px;
    color: var(--primary-fuchsia);
    text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.1);
  }
  
  /* Pop Bounce Animation for Summary Cards */
  .pop-bounce {
    opacity: 0;
    transform: scale(0.5);
    animation: popBounce 0.8s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards;
  }
  .summary-card:nth-child(1) { animation-delay: 1.2s; }
  .summary-card:nth-child(2) { animation-delay: 1.4s; }
  .summary-card:nth-child(3) { animation-delay: 1.6s; }
  
  @keyframes popBounce {
    0% { opacity: 0; transform: scale(0.5); }
    80% { transform: scale(1.1); }
    100% { opacity: 1; transform: scale(1); }
  }
  
  /* --- Guest List Sections --- */
  .attendee-lists h3 {
    font-family: 'Pacifico', cursive;
    color: var(--accent-light);
    margin-bottom: 35px;
    padding-bottom: 15px;
    font-size: 2.5rem;
    font-weight: 400;
    text-align: center;
    text-shadow: 2px 2px 5px rgba(0, 0, 0, 0.3);
    position: relative;
  }
  
  .attendee-lists h3::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 50%;
    transform: translateX(-50%);
    width: 80px;
    height: 3px;
    background-color: var(--accent-gold);
    border-radius: 5px;
    animation: underlineGrow 1s ease forwards;
    animation-delay: 1.8s;
  }
  
  @keyframes underlineGrow {
    from { width: 0; opacity: 0; }
    to { width: 80px; opacity: 1; }
  }
  
  /* Slide In Heading Animation */
  .slide-in-heading {
    opacity: 0;
    transform: translateX(-50px);
    animation: slideInLeft 0.8s ease-out forwards;
    animation-delay: 1s;
  }
  
  @keyframes slideInLeft {
    from { opacity: 0; transform: translateX(-50px); }
    to { opacity: 1; transform: translateX(0); }
  }
  
  /* Category Sections */
  .category {
    margin-bottom: 35px;
    padding: 30px;
    background-color: var(--bg-card);
    border-radius: 15px;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
    border: 1px solid var(--border-card);
    overflow: hidden;
    position: relative;
  }
  
  .category h4 {
    font-family: 'Quicksand', sans-serif;
    color: var(--primary-fuchsia);
    margin-top: 0;
    margin-bottom: 25px;
    padding-bottom: 15px;
    border-bottom: 1px dashed rgba(74, 0, 224, 0.3); /* Dashed blue line */
    font-size: 1.8rem;
    font-weight: 700;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  
  .category-icon {
    margin-right: 15px;
    color: var(--secondary-deep-blue);
    font-size: 1.4em;
  }
  
  .empty-message {
    color: #888;
    font-style: italic;
    text-align: center;
    padding: 20px;
    background-color: var(--accent-light);
    border-radius: 10px;
    border: 1px dashed rgba(231, 0, 138, 0.3);
  }
  
  /* Fade In Text for empty messages */
  .fade-in-text {
    opacity: 0;
    animation: fadeIn 0.8s ease-out forwards;
    animation-delay: 2s; /* Delay after main elements load */
  }
  
  /* Pulse Border for Categories */
  .pulse-border {
    opacity: 0;
    transform: scale(0.95);
    animation: pulseBorder 1s ease-out forwards;
  }
  .category:nth-of-type(1) { animation-delay: 1.8s; }
  .category:nth-of-type(2) { animation-delay: 2.1s; }
  .category:nth-of-type(3) { animation-delay: 2.4s; }
  
  @keyframes pulseBorder {
    0% { opacity: 0; transform: scale(0.95); box-shadow: 0 0 0 0 rgba(231, 0, 138, 0); }
    70% { opacity: 1; transform: scale(1.01); box-shadow: 0 0 0 10px rgba(231, 0, 138, 0.1); }
    100% { opacity: 1; transform: scale(1); box-shadow: 0 0 0 0 rgba(231, 0, 138, 0); }
  }
  
  /* List Items */
  .category ul {
    list-style-type: none;
    padding: 0;
  }
  
  .category li {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 15px 0;
    border-bottom: 1px dashed rgba(74, 0, 224, 0.1);
    color: var(--text-dark);
    font-size: 1.1rem;
    font-weight: 400;
  }
  
  .category li:last-child {
    border-bottom: none;
  }
  
  .remove-btn {
    background-color: var(--primary-fuchsia);
    color: white;
    border: none;
    border-radius: 50%;
    width: 38px;
    height: 38px;
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
    font-size: 1rem;
    transition: background-color 0.3s ease, transform 0.3s ease;
    box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  }
  
  .remove-btn:hover {
    background-color: #c70077; /* Darker fuchsia on hover */
    transform: scale(1.1);
  }
  
  /* Wiggle on Hover for Remove Button */
  .wiggle-on-hover:hover {
    animation: wiggle 0.3s infinite alternate;
  }
  
  @keyframes wiggle {
    0% { transform: rotate(0deg); }
    25% { transform: rotate(5deg); }
    50% { transform: rotate(0deg); }
    75% { transform: rotate(-5deg); }
    100% { transform: rotate(0deg); }
  }
  
  /* Table Styles (18 Candles) */
  .attendee-table-container {
    overflow-x: auto;
    margin-top: 20px;
    border-radius: 10px;
    box-shadow: 0 5px 15px rgba(0, 0, 0, 0.05);
    border: 1px solid rgba(74, 0, 224, 0.1);
  }
  
  .attendee-table {
    width: 100%;
    border-collapse: separate;
    border-spacing: 0;
    background-color: var(--bg-card);
  }
  
  .attendee-table th,
  .attendee-table td {
    padding: 15px 20px;
    border-bottom: 1px solid rgba(74, 0, 224, 0.1);
    border-right: 1px solid rgba(74, 0, 224, 0.1);
    font-family: 'Quicksand', sans-serif;
  }
  
  .attendee-table th:last-child,
  .attendee-table td:last-child {
    border-right: none;
  }
  
  .attendee-table thead th {
    background-color: var(--secondary-deep-blue);
    color: var(--text-light);
    font-weight: 600;
    text-align: left;
    position: sticky;
    top: 0;
    z-index: 1;
  }
  
  .attendee-table tbody tr:nth-child(even) {
    background-color: rgba(255, 255, 255, 0.9);
  }
  
  .attendee-table tbody tr:hover {
    background-color: rgba(231, 0, 138, 0.05); /* Very light fuchsia on hover */
    transform: translateY(-3px);
    box-shadow: 0 5px 15px rgba(0, 0, 0, 0.1);
    transition: background-color 0.2s ease, transform 0.2s ease, box-shadow 0.2s ease;
  }
  
  /* Shimmer Effect for Table */
  .shimmer-effect {
    position: relative;
    overflow: hidden;
  }
  
  .shimmer-effect::after {
    content: '';
    position: absolute;
    top: 0;
    left: -100%;
    width: 100%;
    height: 100%;
    background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.3), transparent);
    animation: shimmer 2.5s infinite;
    animation-delay: 2s; /* Start after table appears */
  }
  
  @keyframes shimmer {
    0% { left: -100%; }
    100% { left: 100%; }
  }
  
  /* List Item Fade In */
  .list-item-fade {
    opacity: 0;
    transform: translateX(-20px);
    animation: fadeInSlide 0.6s ease-out forwards;
    animation-delay: calc(var(--item-index) * 0.08s + 2.5s); /* Stagger with main animation */
  }
  
  @keyframes fadeInSlide {
    from { opacity: 0; transform: translateX(-20px); }
    to { opacity: 1; transform: translateX(0); }
  }
  
  /* Attendance Tags */
  .attendance-tag {
    display: inline-block;
    padding: 8px 15px;
    border-radius: 25px;
    font-weight: 700;
    font-size: 0.95rem;
    letter-spacing: 0.5px;
    text-transform: uppercase;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }
  
  .tag-attended {
    background-color: #3cb371; /* Medium Sea Green */
    color: white;
  }
  
  .tag-pending {
    background-color: #FFA500; /* Orange */
    color: white;
  }
  
  /* Swing In Animation for Tags */
  .swing-in {
    opacity: 0;
    transform: rotateX(-90deg);
    transform-origin: top;
    animation: swingIn 0.8s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards;
    animation-delay: calc(var(--item-index) * 0.08s + 2.8s); /* Stagger after list items */
  }
  
  @keyframes swingIn {
    0% { opacity: 0; transform: rotateX(-90deg); }
    60% { opacity: 1; transform: rotateX(20deg); }
    100% { transform: rotateX(0deg); }
  }
  
  .text-center {
    text-align: center;
  }
  
  /* Responsive Adjustments */
  @media (max-width: 768px) {
    .rsvp-container {
      margin: 20px auto;
      padding: 20px;
    }
  
    h2 {
      font-size: 2.8rem;
    }
  
    .card-grid {
      grid-template-columns: 1fr;
      gap: 15px;
    }
  
    .attendee-lists h3 {
      font-size: 2rem;
    }
  
    .category h4 {
      font-size: 1.5rem;
      flex-direction: column;
      text-align: center;
    }
  
    .category-icon {
      margin-right: 0;
      margin-bottom: 10px;
    }
  
    .attendee-table th,
    .attendee-table td {
      padding: 10px 12px;
      font-size: 0.9rem;
    }
  }
  
  @media (max-width: 480px) {
    h2 {
      font-size: 2.2rem;
    }
  
    .summary-card {
      padding: 15px;
    }
  
    .icon-large {
      font-size: 2.5rem;
    }
  
    .attendee-lists h3 {
      font-size: 1.6rem;
    }
  
    .category {
      padding: 15px;
    }
  
    .category h4 {
      font-size: 1.2rem;
    }
  
    .remove-btn {
      width: 35px;
      height: 35px;
      font-size: 0.9rem;
    }
  }
  </style>