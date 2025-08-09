import { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, getDoc, onSnapshot, collection, getDocs, deleteDoc, runTransaction, query, where, addDoc, serverTimestamp } from 'firebase/firestore';

// Main App component
const App = () => {
  // State for user and Firebase setup
  const [userId, setUserId] = useState(null);
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);

  // State for app functionality
  const [localStream, setLocalStream] = useState(null);
  const [remoteStream, setRemoteStream] = useState(null);
  const [callId, setCallId] = useState(null);
  const [messages, setMessages] = useState([]);
  const [messageInput, setMessageInput] = useState('');
  const [isSearching, setIsSearching] = useState(false);
  const [remoteUserId, setRemoteUserId] = useState(null);
  const [pc, setPc] = useState(null);

  // Refs for video elements
  const localVideoRef = useRef(null);
  const remoteVideoRef = useRef(null);
  const messagesEndRef = useRef(null);

  // Constants
  const servers = {
    iceServers: [
      {
        urls: ['stun:stun1.l.google.com:19302', 'stun:stun2.l.google.com:19302'],
      },
    ],
    iceCandidatePoolSize: 10,
  };

  // Function to initialize Firebase and handle authentication
  useEffect(() => {
    const initializeFirebase = async () => {
      try {
        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const firebaseAuth = getAuth(app);
        const firestoreDb = getFirestore(app);

        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const customToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        
        // Use a persistent user ID for the current user
        let currentUserId;

        // Check if a custom token exists, otherwise sign in anonymously
        if (customToken) {
          await firebaseAuth.signInWithCustomToken(firebaseAuth, customToken);
        } else {
          await signInAnonymously(firebaseAuth);
        }

        onAuthStateChanged(firebaseAuth, (user) => {
          if (user) {
            currentUserId = user.uid;
            setUserId(currentUserId);
            setDb(firestoreDb);
            setAuth(firebaseAuth);
            setIsAuthReady(true);
            console.log('Firebase initialized. User signed in:', user.uid);
          } else {
            console.log('User is signed out.');
            setIsAuthReady(true);
          }
        });
      } catch (error) {
        console.error("Firebase initialization or authentication failed:", error);
      }
    };
    initializeFirebase();
  }, []);

  // Function to get local video stream
  useEffect(() => {
    const getLocalStream = async () => {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
        setLocalStream(stream);
        if (localVideoRef.current) {
          localVideoRef.current.srcObject = stream;
        }
      } catch (err) {
        console.error("Fejl ved adgang til kamera og mikrofon:", err);
      }
    };
    if (isAuthReady) {
      getLocalStream();
    }
  }, [isAuthReady]);

  // Function to start a new chat
  const startChat = async () => {
    if (!db || !userId || !localStream) return;

    setIsSearching(true);
    setRemoteUserId(null);

    // Clean up any old call documents
    await deleteUserCalls();

    try {
      // Look for a waiting user
      const callsCollection = collection(db, `artifacts/${__app_id}/public/data/calls`);
      const waitingCallsQuery = query(callsCollection, where('status', '==', 'waiting'));
      const waitingCallsSnapshot = await getDocs(waitingCallsQuery);
      
      let foundCall = false;

      // Use a transaction to safely update a waiting call
      if (!waitingCallsSnapshot.empty) {
        const waitingCallDoc = waitingCallsSnapshot.docs[0];
        const callRef = doc(db, `artifacts/${__app_id}/public/data/calls`, waitingCallDoc.id);

        try {
          await runTransaction(db, async (transaction) => {
            const callDoc = await transaction.get(callRef);
            if (!callDoc.exists() || callDoc.data().status !== 'waiting') {
              throw new Error("Call document is no longer waiting.");
            }

            // Update call document with new user and status
            transaction.update(callRef, {
              status: 'in-call',
              remoteUserId: userId,
              remoteUserJoinedAt: serverTimestamp(),
            });

            setCallId(callRef.id);
            setRemoteUserId(callDoc.data().userId);
            foundCall = true;
          });
        } catch (e) {
          console.log("Transaction failed or call was taken:", e);
        }
      }

      // If no waiting user was found, create a new call
      if (!foundCall) {
        const newCallRef = doc(callsCollection);
        await setDoc(newCallRef, {
          userId,
          status: 'waiting',
          createdAt: serverTimestamp(),
          offer: null,
          answer: null,
          iceCandidates: []
        });
        setCallId(newCallRef.id);
      }

    } catch (err) {
      console.error("Error finding or creating call:", err);
      setIsSearching(false);
    }
  };

  // Function to hang up the call and start over
  const hangUp = async () => {
    if (pc) {
      pc.close();
      setPc(null);
    }
    if (localStream) {
      localStream.getTracks().forEach(track => track.stop());
    }
    if (remoteVideoRef.current) {
      remoteVideoRef.current.srcObject = null;
    }
    await deleteUserCalls();
    setCallId(null);
    setRemoteStream(null);
    setMessages([]);
    setIsSearching(false);
    setRemoteUserId(null);

    // Re-get local stream for the next call
    const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
    setLocalStream(stream);
    if (localVideoRef.current) {
      localVideoRef.current.srcObject = stream;
    }
  };

  // Function to delete all call documents associated with the current user
  const deleteUserCalls = async () => {
    if (!db || !userId) return;
    try {
      const callsCollection = collection(db, `artifacts/${__app_id}/public/data/calls`);
      const myCallsQuery = query(callsCollection, where('userId', '==', userId));
      const myCallsSnapshot = await getDocs(myCallsQuery);
      myCallsSnapshot.forEach(async (docSnap) => {
        await deleteDoc(doc(callsCollection, docSnap.id));
      });
      const remoteCallsQuery = query(callsCollection, where('remoteUserId', '==', userId));
      const remoteCallsSnapshot = await getDocs(remoteCallsQuery);
      remoteCallsSnapshot.forEach(async (docSnap) => {
         await deleteDoc(doc(callsCollection, docSnap.id));
      });
    } catch (err) {
      console.error("Error deleting calls:", err);
    }
  };

  // Handles the WebRTC signaling and data exchange via Firestore
  useEffect(() => {
    if (!db || !userId || !localStream || !callId) return;

    const setupWebRTC = async () => {
      try {
        const callRef = doc(db, `artifacts/${__app_id}/public/data/calls`, callId);
        const callSnapshot = await getDoc(callRef);
        if (!callSnapshot.exists()) {
          console.error("Call document not found.");
          setIsSearching(false);
          return;
        }

        const callData = callSnapshot.data();
        const isCaller = callData.userId === userId;

        const peerConnection = new RTCPeerConnection(servers);
        setPc(peerConnection);

        // Add local tracks to the peer connection
        localStream.getTracks().forEach(track => {
          peerConnection.addTrack(track, localStream);
        });

        // Event listener for remote stream tracks
        peerConnection.ontrack = (event) => {
          if (remoteVideoRef.current) {
            // Set the remote stream to the video element
            setRemoteStream(event.streams[0]);
            remoteVideoRef.current.srcObject = event.streams[0];
          }
        };

        // Collect ICE candidates and save them to Firestore
        peerConnection.onicecandidate = (event) => {
          if (event.candidate) {
            setDoc(callRef, {
              iceCandidates: [...(callData.iceCandidates || []), event.candidate.toJSON()]
            }, { merge: true });
          }
        };

        // If this user initiated the call (is the caller)
        if (isCaller) {
          console.log("Creating offer...");
          const offerDescription = await peerConnection.createOffer();
          await peerConnection.setLocalDescription(offerDescription);
          await setDoc(callRef, { offer: { sdp: offerDescription.sdp, type: offerDescription.type } }, { merge: true });

          // Listen for the remote user's answer
          const unsubscribe = onSnapshot(callRef, (snapshot) => {
            const data = snapshot.data();
            if (data?.answer && !peerConnection.currentRemoteDescription) {
              console.log("Received answer:", data.answer);
              peerConnection.setRemoteDescription(new RTCSessionDescription(data.answer));
            }
          });
          return unsubscribe;
        } else {
          // This user is joining an existing call (is the answerer)
          console.log("Setting remote offer and creating answer...");
          setRemoteUserId(callData.userId);
          await peerConnection.setRemoteDescription(new RTCSessionDescription(callData.offer));
          const answerDescription = await peerConnection.createAnswer();
          await peerConnection.setLocalDescription(answerDescription);
          await setDoc(callRef, { answer: { sdp: answerDescription.sdp, type: answerDescription.type } }, { merge: true });
        }
        
      } catch (err) {
        console.error("WebRTC setup error:", err);
      } finally {
        setIsSearching(false);
      }
    };
    
    // Listen for changes in the call document to handle WebRTC candidates and status changes
    const callRef = doc(db, `artifacts/${__app_id}/public/data/calls`, callId);
    const unsubscribe = onSnapshot(callRef, (snapshot) => {
      const data = snapshot.data();

      // Start WebRTC connection when both users are in the call
      if (!pc && data?.status === 'in-call') {
        setupWebRTC();
      }

      // Add remote ICE candidates
      if (pc && data?.iceCandidates) {
        data.iceCandidates.forEach(candidate => {
          if (candidate) {
            pc.addIceCandidate(new RTCIceCandidate(candidate));
          }
        });
      }
    });

    // Clean up
    return () => {
      if (unsubscribe) {
        unsubscribe();
      }
    };

  }, [db, userId, localStream, callId]);

  // Handle real-time chat messages
  useEffect(() => {
    if (!db || !callId) return;

    const messagesRef = collection(db, `artifacts/${__app_id}/public/data/calls/${callId}/messages`);
    const unsubscribe = onSnapshot(messagesRef, (snapshot) => {
        const newMessages = snapshot.docs.map(doc => doc.data());
        // Sort messages by timestamp
        newMessages.sort((a, b) => a.timestamp - b.timestamp);
        setMessages(newMessages);
    });

    return () => unsubscribe();
  }, [db, callId]);
  
  // Scrolls to the bottom of the chat box
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  // Function to send a text message
  const sendMessage = async () => {
    if (messageInput.trim() === '' || !db || !callId || !userId) return;

    const messagesRef = collection(db, `artifacts/${__app_id}/public/data/calls/${callId}/messages`);
    await addDoc(messagesRef, {
      senderId: userId,
      message: messageInput,
      timestamp: Date.now(),
    });

    setMessageInput('');
  };

  // UI state for different views
  const renderContent = () => {
    if (!isAuthReady) {
      return <p>Initialiserer...</p>;
    }
    if (isSearching) {
      return (
        <div className="flex flex-col items-center justify-center p-8">
          <div className="w-12 h-12 border-4 border-blue-500 border-t-transparent border-solid rounded-full animate-spin"></div>
          <p className="mt-4 text-gray-400">Søger efter en anden bruger...</p>
          <button onClick={hangUp} className="mt-6 px-6 py-2 bg-red-500 rounded-full font-bold transition-transform transform hover:scale-105">Stop</button>
        </div>
      );
    }
    if (callId && remoteUserId) {
      return (
        <div className="flex flex-col h-full">
          <div className="flex-1 flex flex-col md:flex-row space-y-4 md:space-y-0 md:space-x-4">
            {/* Local video feed */}
            <div className="relative flex-1 bg-gray-800 rounded-xl overflow-hidden shadow-lg">
              <video ref={localVideoRef} autoPlay playsInline muted className="w-full h-full object-cover"></video>
              <div className="absolute top-2 left-2 bg-gray-900/50 text-white text-xs px-2 py-1 rounded-lg">Dig ({userId})</div>
            </div>
            {/* Remote video feed */}
            <div className="relative flex-1 bg-gray-800 rounded-xl overflow-hidden shadow-lg">
              <video ref={remoteVideoRef} autoPlay playsInline className="w-full h-full object-cover"></video>
              {remoteUserId && (
                <div className="absolute top-2 left-2 bg-gray-900/50 text-white text-xs px-2 py-1 rounded-lg">Anden bruger ({remoteUserId})</div>
              )}
            </div>
          </div>
          {/* Chat section */}
          <div className="bg-gray-800 rounded-xl p-4 mt-4 h-[250px] flex flex-col">
            <h3 className="text-lg font-bold mb-2">Chat</h3>
            <div className="flex-1 overflow-y-auto hide-scrollbar">
              {messages.map((msg, index) => (
                <div key={index} className={`flex ${msg.senderId === userId ? 'justify-end' : 'justify-start'}`}>
                  <div className={`p-2 my-1 rounded-xl max-w-[75%] ${msg.senderId === userId ? 'bg-blue-600 text-white' : 'bg-gray-700 text-white'}`}>
                    <p className="text-sm">{msg.message}</p>
                  </div>
                </div>
              ))}
              <div ref={messagesEndRef} />
            </div>
            <div className="mt-2 flex">
              <input 
                type="text" 
                value={messageInput} 
                onChange={(e) => setMessageInput(e.target.value)} 
                onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
                placeholder="Skriv en besked..." 
                className="flex-1 bg-gray-700 text-white rounded-l-xl p-2 focus:outline-none"
              />
              <button 
                onClick={sendMessage} 
                className="bg-green-500 hover:bg-green-600 text-white rounded-r-xl px-4 py-2 transition-colors duration-200"
              >
                Send
              </button>
            </div>
          </div>
          <button onClick={hangUp} className="mt-6 px-6 py-3 bg-red-500 rounded-full font-bold transition-transform transform hover:scale-105">Stop chat</button>
        </div>
      );
    }
    return (
      <div className="flex flex-col items-center justify-center p-8">
        <p className="text-center text-gray-300 text-xl">
          Velkommen til Connectify. Start en chat med en tilfældig person.
        </p>
        <button onClick={startChat} className="mt-6 px-8 py-3 bg-blue-500 rounded-full font-bold transition-transform transform hover:scale-105">Start chat</button>
      </div>
    );
  };

  return (
    <div className="bg-gray-950 text-white min-h-screen flex items-start justify-center p-4 font-inter">
      <div className="w-full max-w-4xl mx-auto py-8">
        <h1 className="text-5xl font-extrabold text-center mb-10 text-transparent bg-clip-text bg-gradient-to-r from-green-400 to-blue-500">Connectify</h1>
        <div className="bg-gray-900 rounded-3xl shadow-2xl p-6 md:p-8">
          {renderContent()}
        </div>
        <p className="text-center text-gray-500 mt-8 text-sm">Din bruger ID: {userId || 'Ingen'}</p>
      </div>
    </div>
  );
};

export default App;
