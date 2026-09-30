# Part 083: WebRTC

## WebRTC คืออะไร?

WebRTC (Web Real-Time Communication) คือ open-source project และ API standard ที่ช่วยให้ browser และ mobile apps สามารถสื่อสาร real-time audio, video, และ data โดยตรงระหว่างกัน (peer-to-peer) โดยไม่ต้องผ่าน server กลาง

---

## 1. WebRTC Basics

### 1.1 Architecture Overview

```
Caller                    Signaling Server              Callee
  |                            |                           |
  |--- Create Offer ---------->|                           |
  |                            |--- Forward Offer -------->|
  |                            |<-- Answer ----------------| 
  |<--- Forward Answer --------|                           |
  |<--- ICE Candidates ------->|<--- ICE Candidates ------>|
  |                                                        |
  |<============ Direct P2P Connection ===================>|
         (Audio/Video/Data streams)
```

### 1.2 สิ่งที่ต้องรู้

- **SDP** (Session Description Protocol) - ข้อมูลเกี่ยวกับ media capabilities
- **ICE** (Interactive Connectivity Establishment) - หา network path ระหว่าง peers
- **STUN/TURN** - help peers ผ่าน NAT/firewall
- **RTCPeerConnection** - จัดการ P2P connection

---

## 2. ติดตั้ง react-native-webrtc

### 2.1 Installation

```bash
npm install react-native-webrtc

# iOS
cd ios && pod install

# Android - ต้องเพิ่ม permissions ใน AndroidManifest.xml
```

### 2.2 Android Manifest

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.CHANGE_NETWORK_STATE" />
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
```

### 2.3 iOS Info.plist

```xml
<!-- ios/YourApp/Info.plist -->
<key>NSCameraUsageDescription</key>
<string>ต้องการใช้กล้องสำหรับ video call</string>
<key>NSMicrophoneUsageDescription</key>
<string>ต้องการใช้ไมค์สำหรับ voice call</string>
```

---

## 3. Peer Connection Setup

### 3.1 WebRTC Service

```typescript
// services/WebRTCService.ts
import {
  RTCPeerConnection,
  RTCIceCandidate,
  RTCSessionDescription,
  MediaStream,
  mediaDevices,
  RTCView,
  MediaStreamTrack
} from 'react-native-webrtc';

const ICE_SERVERS = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
    // TURN server (ถ้าต้องการ)
    // {
    //   urls: 'turn:your-turn-server.com:3478',
    //   username: 'user',
    //   credential: 'password'
    // }
  ]
};

export class WebRTCService {
  private peerConnection: RTCPeerConnection | null = null;
  private localStream: MediaStream | null = null;
  private remoteStream: MediaStream | null = null;
  
  // Callbacks
  onLocalStream?: (stream: MediaStream) => void;
  onRemoteStream?: (stream: MediaStream) => void;
  onIceCandidate?: (candidate: RTCIceCandidate) => void;
  onConnectionStateChange?: (state: string) => void;
  onCallEnded?: () => void;
  
  async initialize() {
    this.peerConnection = new RTCPeerConnection(ICE_SERVERS);
    
    // Handle remote stream
    this.peerConnection.ontrack = ({ streams }) => {
      if (streams && streams[0]) {
        this.remoteStream = streams[0];
        this.onRemoteStream?.(streams[0]);
      }
    };
    
    // Handle ICE candidates
    this.peerConnection.onicecandidate = ({ candidate }) => {
      if (candidate) {
        this.onIceCandidate?.(candidate);
      }
    };
    
    // Handle connection state
    this.peerConnection.onconnectionstatechange = () => {
      const state = this.peerConnection?.connectionState ?? '';
      this.onConnectionStateChange?.(state);
      
      if (state === 'disconnected' || state === 'failed' || state === 'closed') {
        this.onCallEnded?.();
      }
    };
    
    // Handle ICE connection state
    this.peerConnection.oniceconnectionstatechange = () => {
      console.log('ICE state:', this.peerConnection?.iceConnectionState);
    };
  }
  
  async getLocalStream(video = true, audio = true): Promise<MediaStream> {
    const constraints = {
      audio: audio ? {
        echoCancellation: true,
        noiseSuppression: true,
        autoGainControl: true
      } : false,
      video: video ? {
        facingMode: 'user',
        width: { ideal: 1280 },
        height: { ideal: 720 },
        frameRate: { ideal: 30 }
      } : false
    };
    
    this.localStream = await mediaDevices.getUserMedia(constraints);
    this.onLocalStream?.(this.localStream);
    
    // Add tracks to peer connection
    this.localStream.getTracks().forEach(track => {
      this.peerConnection?.addTrack(track, this.localStream!);
    });
    
    return this.localStream;
  }
  
  async createOffer(): Promise<RTCSessionDescription> {
    if (!this.peerConnection) throw new Error('Not initialized');
    
    const offer = await this.peerConnection.createOffer({
      offerToReceiveAudio: true,
      offerToReceiveVideo: true
    });
    
    await this.peerConnection.setLocalDescription(offer);
    return offer;
  }
  
  async createAnswer(offer: RTCSessionDescription): Promise<RTCSessionDescription> {
    if (!this.peerConnection) throw new Error('Not initialized');
    
    await this.peerConnection.setRemoteDescription(
      new RTCSessionDescription(offer)
    );
    
    const answer = await this.peerConnection.createAnswer();
    await this.peerConnection.setLocalDescription(answer);
    
    return answer;
  }
  
  async setRemoteAnswer(answer: RTCSessionDescription) {
    if (!this.peerConnection) throw new Error('Not initialized');
    await this.peerConnection.setRemoteDescription(
      new RTCSessionDescription(answer)
    );
  }
  
  async addIceCandidate(candidate: RTCIceCandidate) {
    if (!this.peerConnection) throw new Error('Not initialized');
    await this.peerConnection.addIceCandidate(
      new RTCIceCandidate(candidate)
    );
  }
  
  toggleMute(): boolean {
    const audioTrack = this.localStream?.getAudioTracks()[0];
    if (audioTrack) {
      audioTrack.enabled = !audioTrack.enabled;
      return audioTrack.enabled;
    }
    return false;
  }
  
  toggleCamera(): boolean {
    const videoTrack = this.localStream?.getVideoTracks()[0];
    if (videoTrack) {
      videoTrack.enabled = !videoTrack.enabled;
      return videoTrack.enabled;
    }
    return false;
  }
  
  async switchCamera() {
    const videoTrack = this.localStream?.getVideoTracks()[0] as any;
    if (videoTrack && videoTrack._switchCamera) {
      videoTrack._switchCamera();
    }
  }
  
  endCall() {
    // Stop local stream tracks
    this.localStream?.getTracks().forEach(track => track.stop());
    this.localStream = null;
    
    // Close peer connection
    this.peerConnection?.close();
    this.peerConnection = null;
    
    this.remoteStream = null;
  }
  
  getLocalStream(): MediaStream | null {
    return this.localStream;
  }
  
  getRemoteStream(): MediaStream | null {
    return this.remoteStream;
  }
}
```

---

## 4. Signaling Server

### 4.1 Socket.io Signaling

```typescript
// services/SignalingService.ts
import { io, Socket } from 'socket.io-client';

interface SignalingEvents {
  offer: { from: string; offer: any };
  answer: { from: string; answer: any };
  iceCandidate: { from: string; candidate: any };
  userJoined: { userId: string };
  userLeft: { userId: string };
  callEnded: void;
}

export class SignalingService {
  private socket: Socket | null = null;
  private roomId: string = '';
  private userId: string = '';
  
  onOffer?: (from: string, offer: any) => void;
  onAnswer?: (from: string, answer: any) => void;
  onIceCandidate?: (from: string, candidate: any) => void;
  onUserJoined?: (userId: string) => void;
  onUserLeft?: (userId: string) => void;
  onCallEnded?: () => void;
  
  connect(serverUrl: string, userId: string) {
    this.userId = userId;
    
    this.socket = io(serverUrl, {
      transports: ['websocket'],
      reconnection: true,
      reconnectionAttempts: 5,
      reconnectionDelay: 1000
    });
    
    this.setupListeners();
  }
  
  private setupListeners() {
    if (!this.socket) return;
    
    this.socket.on('offer', ({ from, offer }) => {
      this.onOffer?.(from, offer);
    });
    
    this.socket.on('answer', ({ from, answer }) => {
      this.onAnswer?.(from, answer);
    });
    
    this.socket.on('ice-candidate', ({ from, candidate }) => {
      this.onIceCandidate?.(from, candidate);
    });
    
    this.socket.on('user-joined', ({ userId }) => {
      this.onUserJoined?.(userId);
    });
    
    this.socket.on('user-left', ({ userId }) => {
      this.onUserLeft?.(userId);
    });
    
    this.socket.on('call-ended', () => {
      this.onCallEnded?.();
    });
  }
  
  joinRoom(roomId: string) {
    this.roomId = roomId;
    this.socket?.emit('join-room', { roomId, userId: this.userId });
  }
  
  sendOffer(to: string, offer: any) {
    this.socket?.emit('offer', { to, from: this.userId, offer });
  }
  
  sendAnswer(to: string, answer: any) {
    this.socket?.emit('answer', { to, from: this.userId, answer });
  }
  
  sendIceCandidate(to: string, candidate: any) {
    this.socket?.emit('ice-candidate', { to, from: this.userId, candidate });
  }
  
  endCall() {
    this.socket?.emit('end-call', { roomId: this.roomId });
  }
  
  leaveRoom() {
    this.socket?.emit('leave-room', { roomId: this.roomId });
  }
  
  disconnect() {
    this.socket?.disconnect();
    this.socket = null;
  }
}
```

---

## 5. Workshop: Video Chat App

### Step 1: Video Call Hook

```typescript
// hooks/useVideoCall.ts
import { useState, useEffect, useCallback, useRef } from 'react';
import { Alert, Platform } from 'react-native';
import { PermissionsAndroid } from 'react-native';
import { WebRTCService } from '../services/WebRTCService';
import { SignalingService } from '../services/SignalingService';
import type { MediaStream } from 'react-native-webrtc';

interface CallState {
  isInitialized: boolean;
  isInCall: boolean;
  isMuted: boolean;
  isCameraOff: boolean;
  connectionState: string;
  localStream: MediaStream | null;
  remoteStream: MediaStream | null;
  remoteUserId: string | null;
}

export function useVideoCall(userId: string, roomId: string) {
  const webRTC = useRef(new WebRTCService()).current;
  const signaling = useRef(new SignalingService()).current;
  
  const [state, setState] = useState<CallState>({
    isInitialized: false,
    isInCall: false,
    isMuted: false,
    isCameraOff: false,
    connectionState: 'new',
    localStream: null,
    remoteStream: null,
    remoteUserId: null
  });
  
  const requestPermissions = async (): Promise<boolean> => {
    if (Platform.OS === 'android') {
      const granted = await PermissionsAndroid.requestMultiple([
        PermissionsAndroid.PERMISSIONS.CAMERA,
        PermissionsAndroid.PERMISSIONS.RECORD_AUDIO
      ]);
      
      return (
        granted[PermissionsAndroid.PERMISSIONS.CAMERA] === 'granted' &&
        granted[PermissionsAndroid.PERMISSIONS.RECORD_AUDIO] === 'granted'
      );
    }
    return true;
  };
  
  const initialize = useCallback(async () => {
    const hasPermissions = await requestPermissions();
    if (!hasPermissions) {
      Alert.alert('Permission denied', 'Need camera and microphone permissions');
      return;
    }
    
    // Connect to signaling server
    signaling.connect('https://your-signaling-server.com', userId);
    
    // Setup webrtc callbacks
    webRTC.onLocalStream = (stream) => {
      setState(prev => ({ ...prev, localStream: stream }));
    };
    
    webRTC.onRemoteStream = (stream) => {
      setState(prev => ({ ...prev, remoteStream: stream }));
    };
    
    webRTC.onIceCandidate = (candidate) => {
      if (state.remoteUserId) {
        signaling.sendIceCandidate(state.remoteUserId, candidate);
      }
    };
    
    webRTC.onConnectionStateChange = (connectionState) => {
      setState(prev => ({ ...prev, connectionState }));
    };
    
    webRTC.onCallEnded = () => {
      handleEndCall();
    };
    
    // Setup signaling callbacks
    signaling.onUserJoined = async (joinedUserId) => {
      setState(prev => ({ ...prev, remoteUserId: joinedUserId }));
      // Caller creates offer
      await startCall(joinedUserId);
    };
    
    signaling.onOffer = async (from, offer) => {
      setState(prev => ({ ...prev, remoteUserId: from }));
      await handleOffer(from, offer);
    };
    
    signaling.onAnswer = async (from, answer) => {
      await webRTC.setRemoteAnswer(answer);
    };
    
    signaling.onIceCandidate = async (from, candidate) => {
      await webRTC.addIceCandidate(candidate);
    };
    
    signaling.onCallEnded = handleEndCall;
    
    // Initialize webrtc
    await webRTC.initialize();
    await webRTC.getLocalStream();
    
    // Join room
    signaling.joinRoom(roomId);
    
    setState(prev => ({ ...prev, isInitialized: true }));
  }, [userId, roomId]);
  
  const startCall = async (remoteUserId: string) => {
    try {
      const offer = await webRTC.createOffer();
      signaling.sendOffer(remoteUserId, offer);
      setState(prev => ({ ...prev, isInCall: true }));
    } catch (error) {
      console.error('Failed to start call:', error);
    }
  };
  
  const handleOffer = async (from: string, offer: any) => {
    try {
      const answer = await webRTC.createAnswer(offer);
      signaling.sendAnswer(from, answer);
      setState(prev => ({ ...prev, isInCall: true }));
    } catch (error) {
      console.error('Failed to handle offer:', error);
    }
  };
  
  const handleEndCall = useCallback(() => {
    webRTC.endCall();
    setState(prev => ({
      ...prev,
      isInCall: false,
      remoteStream: null,
      remoteUserId: null,
      connectionState: 'closed'
    }));
  }, [webRTC]);
  
  const endCall = useCallback(() => {
    signaling.endCall();
    handleEndCall();
  }, [signaling, handleEndCall]);
  
  const toggleMute = useCallback(() => {
    const isEnabled = webRTC.toggleMute();
    setState(prev => ({ ...prev, isMuted: !isEnabled }));
  }, [webRTC]);
  
  const toggleCamera = useCallback(() => {
    const isEnabled = webRTC.toggleCamera();
    setState(prev => ({ ...prev, isCameraOff: !isEnabled }));
  }, [webRTC]);
  
  const switchCamera = useCallback(() => {
    webRTC.switchCamera();
  }, [webRTC]);
  
  useEffect(() => {
    return () => {
      webRTC.endCall();
      signaling.disconnect();
    };
  }, []);
  
  return {
    ...state,
    initialize,
    endCall,
    toggleMute,
    toggleCamera,
    switchCamera
  };
}
```

### Step 2: Video Call Screen

```tsx
// screens/VideoCallScreen.tsx
import React, { useEffect } from 'react';
import {
  View,
  StyleSheet,
  TouchableOpacity,
  Text,
  StatusBar,
  SafeAreaView
} from 'react-native';
import { RTCView } from 'react-native-webrtc';
import { useVideoCall } from '../hooks/useVideoCall';

interface Props {
  route: {
    params: {
      userId: string;
      roomId: string;
      contactName: string;
    };
  };
  navigation: any;
}

const VideoCallScreen: React.FC<Props> = ({ route, navigation }) => {
  const { userId, roomId, contactName } = route.params;
  
  const {
    isInitialized,
    isInCall,
    isMuted,
    isCameraOff,
    connectionState,
    localStream,
    remoteStream,
    initialize,
    endCall,
    toggleMute,
    toggleCamera,
    switchCamera
  } = useVideoCall(userId, roomId);
  
  useEffect(() => {
    initialize();
  }, []);
  
  const handleEndCall = () => {
    endCall();
    navigation.goBack();
  };
  
  const getConnectionStatus = () => {
    switch (connectionState) {
      case 'new': return 'กำลังเชื่อมต่อ...';
      case 'connecting': return 'กำลังเชื่อมต่อ...';
      case 'connected': return 'เชื่อมต่อแล้ว';
      case 'disconnected': return 'ขาดการเชื่อมต่อ';
      case 'failed': return 'เชื่อมต่อล้มเหลว';
      case 'closed': return 'ปิดการเชื่อมต่อ';
      default: return connectionState;
    }
  };
  
  return (
    <View style={styles.container}>
      <StatusBar barStyle="light-content" backgroundColor="#000" />
      
      {/* Remote Video (Full Screen) */}
      {remoteStream ? (
        <RTCView
          streamURL={remoteStream.toURL()}
          style={styles.remoteVideo}
          objectFit="cover"
          mirror={false}
        />
      ) : (
        <View style={styles.waitingContainer}>
          <Text style={styles.waitingText}>{getConnectionStatus()}</Text>
          <Text style={styles.contactName}>{contactName}</Text>
        </View>
      )}
      
      {/* Local Video (PiP) */}
      {localStream && (
        <TouchableOpacity
          style={styles.localVideoContainer}
          onPress={switchCamera}
        >
          <RTCView
            streamURL={localStream.toURL()}
            style={styles.localVideo}
            objectFit="cover"
            mirror={true}
          />
          {isCameraOff && (
            <View style={styles.cameraOffOverlay}>
              <Text style={styles.cameraOffText}>กล้องปิด</Text>
            </View>
          )}
        </TouchableOpacity>
      )}
      
      {/* Controls */}
      <SafeAreaView style={styles.controlsContainer}>
        <View style={styles.controls}>
          {/* Mute Button */}
          <TouchableOpacity
            style={[styles.controlButton, isMuted && styles.activeControl]}
            onPress={toggleMute}
          >
            <Text style={styles.controlIcon}>{isMuted ? '🔇' : '🎤'}</Text>
            <Text style={styles.controlLabel}>{isMuted ? 'เปิด' : 'ปิด'}เสียง</Text>
          </TouchableOpacity>
          
          {/* End Call Button */}
          <TouchableOpacity
            style={[styles.controlButton, styles.endCallButton]}
            onPress={handleEndCall}
          >
            <Text style={styles.controlIcon}>📵</Text>
            <Text style={styles.controlLabel}>วางสาย</Text>
          </TouchableOpacity>
          
          {/* Camera Toggle */}
          <TouchableOpacity
            style={[styles.controlButton, isCameraOff && styles.activeControl]}
            onPress={toggleCamera}
          >
            <Text style={styles.controlIcon}>{isCameraOff ? '📷' : '📸'}</Text>
            <Text style={styles.controlLabel}>{isCameraOff ? 'เปิด' : 'ปิด'}กล้อง</Text>
          </TouchableOpacity>
        </View>
      </SafeAreaView>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#000'
  },
  remoteVideo: {
    flex: 1
  },
  waitingContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#1a1a2e'
  },
  waitingText: {
    color: '#fff',
    fontSize: 18,
    marginBottom: 12
  },
  contactName: {
    color: '#fff',
    fontSize: 28,
    fontWeight: 'bold'
  },
  localVideoContainer: {
    position: 'absolute',
    top: 60,
    right: 16,
    width: 120,
    height: 160,
    borderRadius: 12,
    overflow: 'hidden',
    borderWidth: 2,
    borderColor: '#fff'
  },
  localVideo: {
    flex: 1
  },
  cameraOffOverlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: '#333',
    justifyContent: 'center',
    alignItems: 'center'
  },
  cameraOffText: {
    color: '#fff',
    fontSize: 12
  },
  controlsContainer: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0
  },
  controls: {
    flexDirection: 'row',
    justifyContent: 'space-evenly',
    paddingVertical: 20,
    paddingHorizontal: 20,
    backgroundColor: 'rgba(0,0,0,0.6)'
  },
  controlButton: {
    alignItems: 'center',
    padding: 12,
    borderRadius: 12,
    minWidth: 70,
    backgroundColor: 'rgba(255,255,255,0.15)'
  },
  activeControl: {
    backgroundColor: 'rgba(255,59,48,0.4)'
  },
  endCallButton: {
    backgroundColor: '#FF3B30'
  },
  controlIcon: {
    fontSize: 28,
    marginBottom: 4
  },
  controlLabel: {
    color: '#fff',
    fontSize: 11
  }
});

export default VideoCallScreen;
```

---

## Tips สำหรับ WebRTC

1. **STUN/TURN servers** - ต้องใช้สำหรับ production เพื่อเจาะ NAT
2. **ICE candidate gathering** - รับ candidates ทั้งหมดก่อน connect
3. **Bitrate adaptation** - adjust quality ตาม network
4. **Error handling** - handle connection failures gracefully
5. **Battery life** - WebRTC ใช้พลังงานมาก minimize เมื่อ background

## สรุป

WebRTC ใน React Native ช่วยสร้าง:
1. Video calling apps
2. Voice calls
3. Screen sharing
4. Real-time data exchange
5. Multi-party calls (with media servers)
