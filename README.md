import express from 'express';
import http from 'http';
import { Server } from 'socket.io';
import cors from 'cors';
const app = express();
app.use(cors());
const server = http.createServer(app);
const io = new Server(server, {
    cors: {
        origin: "*", 
        methods: ["GET", "POST"]
    }
});
interface Player {
    id: string;
    position: { x: number, y: number, z: number };
    rotation: { x: number, y: number, z: number };
    color: string;
}
const players: Record<string, Player> = {};
io.on('connection', (socket) => {
    console.log('User connected:', socket.id);
    // Initial state
    players[socket.id] = {
        id: socket.id,
        position: { x: 0, y: 1, z: 0 },
        rotation: { x: 0, y: 0, z: 0 },
        color: '#' + Math.floor(Math.random() * 16777215).toString(16)
    };
    socket.emit('currentPlayers', players);
    socket.broadcast.emit('newPlayer', players[socket.id]);
    socket.on('playerMove', (data) => {
        if (players[socket.id]) {
            players[socket.id].position = data.position;
            players[socket.id].rotation = data.rotation;
            socket.broadcast.emit('playerMoved', players[socket.id]);
        }
    });
    socket.on('shoot', () => {
        socket.broadcast.emit('playerShot', { id: socket.id });
    });
    socket.on('disconnect', () => {
        console.log('User disconnected:', socket.id);
        delete players[socket.id];
        io.emit('playerDisconnected', socket.id);
    });
});
const PORT = 3001;
server.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});import { create } from 'zustand';
import { io, Socket } from 'socket.io-client';
// Connect to environment variable URL or default to local/LAN
const socketUrl = import.meta.env.VITE_SERVER_URL || `http://${window.location.hostname}:3001`;
export const socket = io(socketUrl, {
    autoConnect: true,
    transports: ['websocket', 'polling'] 
});
interface PlayerState {
    id: string;
    position: { x: number; y: number; z: number };
    rotation: { x: number; y: number; z: number };
    color: string;
}
interface GameStore {
    players: Record<string, PlayerState>;
    isOffline: boolean;            // From previous updates
    setOffline: (v: boolean) => void;
    stats: any; 
    setPlayers: (players: Record<string, PlayerState>) => void;
    addPlayer: (player: PlayerState) => void;
    updatePlayer: (player: PlayerState) => void;
    removePlayer: (id: string) => void;
    addStat: (key: string) => void;
}
export const useGameStore = create<GameStore>((set) => ({
    players: {},
    isOffline: false,
    stats: { kills: 0, deaths: 0, shotsFired: 0, shotsHit: 0 },
    setOffline: (v) => set({ isOffline: v }),
    setPlayers: (players) => set({ players }),
    addPlayer: (player) => set((state) => ({ players: { ...state.players, [player.id]: player } })),
    updatePlayer: (player) => set((state) => ({
        players: { ...state.players, [player.id]: { ...state.players[player.id], ...player } }
    })),
    removePlayer: (id) => set((state) => {
        const newPlayers = { ...state.players };
        delete newPlayers[id];
        return { players: newPlayers };
    }),
    addStat: (key) => set((state) => ({ 
        stats: { ...state.stats, [key]: (state.stats[key] || 0) + 1 } 
    }))
}));
// Socket listeners
socket.on('currentPlayers', (players: Record<string, PlayerState>) => {
    useGameStore.getState().setPlayers(players);
});
socket.on('newPlayer', (player: PlayerState) => {
    useGameStore.getState().addPlayer(player);
});
socket.on('playerMoved', (player: PlayerState) => {
    useGameStore.getState().updatePlayer(player);
});
socket.on('playerDisconnected', (id: string) => {
    useGameStore.getState().removePlayer(id);
});import { useEffect, useRef, useState } from 'react';
import { useFrame } from '@react-three/fiber';
import { RigidBody, CapsuleCollider, RapierRigidBody } from '@react-three/rapier';
import { PerspectiveCamera, PointerLockControls } from '@react-three/drei';
import { Vector3, MathUtils } from 'three';
import { socket, useGameStore } from '../store/gameStore';
const SPEED = 5;
const JUMP_FORCE = 5;
const ADS_FOV = 40;
const REGULAR_FOV = 75;
export function LocalPlayer() {
    const rigidBody = useRef<RapierRigidBody>(null);
    const [keys, setKeys] = useState({ w: false, a: false, s: false, d: false, space: false });
    const [isADS, setIsADS] = useState(false);
    const lastUpdate = useRef(0);
    const addStat = useGameStore(state => state.addStat);
    useEffect(() => {
        const handleKeyDown = (e: KeyboardEvent) => {
            switch (e.key.toLowerCase()) {
                case 'w': setKeys(k => ({ ...k, w: true })); break;
                case 'a': setKeys(k => ({ ...k, a: true })); break;
                case 's': setKeys(k => ({ ...k, s: true })); break;
                case 'd': setKeys(k => ({ ...k, d: true })); break;
                case ' ': setKeys(k => ({ ...k, space: true })); break;
            }
        };
        const handleKeyUp = (e: KeyboardEvent) => {
            switch (e.key.toLowerCase()) {
                case 'w': setKeys(k => ({ ...k, w: false })); break;
                case 'a': setKeys(k => ({ ...k, a: false })); break;
                case 's': setKeys(k => ({ ...k, s: false })); break;
                case 'd': setKeys(k => ({ ...k, d: false })); break;
                case ' ': setKeys(k => ({ ...k, space: false })); break;
            }
        };
        // Use generic casting or proper typing in a real editor
        window.addEventListener('keydown', handleKeyDown);
        window.addEventListener('keyup', handleKeyUp);
        const handleMouseDown = (e: MouseEvent) => {
            if (e.button === 0) { // Left Click
                socket.emit('shoot');
                addStat('shotsFired');
            } else if (e.button === 2) { // Right Click
                setIsADS(true);
            }
        };
        const handleMouseUp = (e: MouseEvent) => {
            if (e.button === 2) setIsADS(false);
        };
        window.addEventListener('mousedown', handleMouseDown);
        window.addEventListener('mouseup', handleMouseUp);
        return () => {
            window.removeEventListener('keydown', handleKeyDown);
            window.removeEventListener('keyup', handleKeyUp);
            window.removeEventListener('mousedown', handleMouseDown);
            window.removeEventListener('mouseup', handleMouseUp);
        };
    }, []);
    useFrame((state, delta) => {
        if (!rigidBody.current) return;
        // ADS Lerp
        const targetFOV = isADS ? ADS_FOV : REGULAR_FOV;
        state.camera.fov = MathUtils.lerp(state.camera.fov, targetFOV, delta * 10);
        state.camera.updateProjectionMatrix();
        // Movement Logic
        const { w, a, s, d, space } = keys;
        const linvel = rigidBody.current.linvel();
        const camera = state.camera;
        const forward = new Vector3(0, 0, -1).applyQuaternion(camera.quaternion);
        forward.y = 0; forward.normalize();
        const right = new Vector3(1, 0, 0).applyQuaternion(camera.quaternion);
        right.y = 0; right.normalize();
        const direction = new Vector3(0, 0, 0);
        if (w) direction.add(forward);
        if (s) direction.sub(forward);
        if (d) direction.add(right);
        if (a) direction.sub(right);
        if (direction.length() > 0) {
            direction.normalize().multiplyScalar(SPEED);
            rigidBody.current.setLinvel({ x: direction.x, y: linvel.y, z: direction.z }, true);
        } else {
            rigidBody.current.setLinvel({ x: 0, y: linvel.y, z: 0 }, true);
        }
        if (space && Math.abs(linvel.y) < 0.1) {
            rigidBody.current.applyImpulse({ x: 0, y: JUMP_FORCE, z: 0 }, true);
        }
        // Network Sync
        const now = Date.now();
        if (now - lastUpdate.current > 20) {
            const pos = rigidBody.current.translation();
            const rot = camera.rotation;
            socket.emit('playerMove', {
                position: pos,
                rotation: { x: rot.x, y: rot.y, z: rot.z }
            });
            lastUpdate.current = now;
        }
    });
    return (
        <group>
            <RigidBody ref={rigidBody} colliders={false} mass={1} type="dynamic" position={[0, 5, 0]} enabledRotations={[false, false, false]}>
                <CapsuleCollider args={[1, 0.3]} />
            </RigidBody>
            <GroupFollower targetRef={rigidBody} offset={[0, 1.5, 0]}>
                <PerspectiveCamera makeDefault />
            </GroupFollower>
            <PointerLockControls />
        </group>
    );
}
function GroupFollower({ targetRef, offset, children }: any) {
    const group = useRef<any>();
    useFrame(() => {
        if (!targetRef.current || !group.current) return;
        const t = targetRef.current.translation();
        group.current.position.set(t.x + offset[0], t.y + offset[1], t.z + offset[2]);
    });
    return <group ref={group}>{children}</group>;
}import { Canvas } from '@react-three/fiber';
import { Experience } from './components/Experience';
import { useGameStore, socket } from './store/gameStore';
function App() {
    const players = useGameStore((state) => state.players);
    const isOffline = useGameStore((state) => state.isOffline);
    const setOffline = useGameStore((state) => state.setOffline);
    const myId = socket.id;
    return (
        <>
            <Canvas shadows camera={{ fov: 75, position: [0, 5, 10] }}>
                <color attach="background" args={['#000000']} />
                <Experience />
            </Canvas>
            <div id="ui">
                <div className="crosshair"></div>
                <div style={{ position: 'absolute', top: 20, left: 20 }}>
                    <h3>Controls: WASD to Move, SPACE to Jump, Mouse to Look</h3>
                    <p>RIGHT CLICK to Aim (ADS) | LEFT CLICK to Shoot</p>
                    
                    <div style={{ marginBottom: 15 }}>
                        <button 
                            onClick={() => setOffline(!isOffline)}
                            style={{ 
                                padding: '10px 20px', 
                                background: isOffline ? '#e74c3c' : '#2ecc71', 
                                color: 'white', 
                                border: 'none', 
                                borderRadius: '4px',
                                fontWeight: 'bold'
                            }}
                        >
                            MODE: {isOffline ? 'OFFLINE (Bot Training)' : 'ONLINE MULTIPLAYER'}
                        </button>
                    </div>
                    {!isOffline && <p>Connected Players: {Object.keys(players).length}</p>}
                    <p style={{ opacity: 0.5, fontSize: '12px' }}>ID: {myId}</p>
                </div>
            </div>
        </>
    );
}
export default App;
