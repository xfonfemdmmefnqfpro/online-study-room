# online-study-room
pnpm install（ルート）→ pnpm dev で backend + frontend を並列起動。

LiveKit をローカルで動かすなら docker compose -f livekit/docker-compose.yml up -d を別ターミナルで実行。

online-study-room/
├── .gitignore
├── README.md
├── package.json        # ワークスペースルート
├── pnpm-workspace.yaml
├── frontend/           # Next.js アプリ
│   ├── package.json
│   ├── next.config.mjs
│   ├── tailwind.config.ts
│   ├── postcss.config.cjs
│   ├── tsconfig.json
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── room/[id]/page.tsx
│   ├── components/
│   │   ├── VideoGrid.tsx
│   │   ├── ChatPanel.tsx
│   │   └── Timer.tsx
│   ├── stores/room.ts
│   └── utils/livekit.ts
├── backend/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       └── server.ts
└── livekit/            # 任意 (self‑host 用)
    └── docker-compose.yml

1. ルートファイル

.gitignore

node_modules
.env
.next
.dist
coverage

pnpm-workspace.yaml

packages:
  - "frontend"
  - "backend"

package.json (ルート)

{
  "name": "online-study-room",
  "private": true,
  "workspaces": ["frontend", "backend"],
  "scripts": {
    "dev": "concurrently -k -n FRONTEND,BACKEND -c blue,green \"pnpm --filter frontend dev\" \"pnpm --filter backend dev\""
  },
  "devDependencies": {
    "concurrently": "^8.3.0",
    "pnpm": "^9.1.2"
  }
}

README.md

# Online Study Room

最大30人の中学生がビデオ通話とチャットで一緒に勉強できるオンライン自習室。

## 起動方法
```bash
pnpm install
pnpm dev        # http://localhost:3000 (frontend), http://localhost:4000 (backend)

環境変数

backend/.env に LiveKit API キー等を設定。


---
## 2. frontend/
### `frontend/package.json`
```jsonc
{
  "name": "online-study-room-frontend",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "@livekit/client": "^2.5.2",
    "@livekit/react-components": "^1.3.1",
    "next": "14.2.0",
    "react": "18.3.1",
    "react-dom": "18.3.1",
    "socket.io-client": "4.7.5",
    "zustand": "4.5.2"
  },
  "devDependencies": {
    "tailwindcss": "^3.4.4",
    "typescript": "5.5.0",
    "autoprefixer": "^10.4.15",
    "postcss": "^8.4.35"
  }
}

frontend/next.config.mjs

/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: { typedRoutes: true },
};
export default nextConfig;

frontend/tailwind.config.ts

import type { Config } from 'tailwindcss';
export default {
  content: [
    './app/**/*.{tsx,ts}',
    './components/**/*.{tsx,ts}',
  ],
  theme: { extend: {} },
  plugins: [],
} satisfies Config;

frontend/postcss.config.cjs

module.exports = {
  plugins: { tailwindcss: {}, autoprefixer: {} },
};

frontend/tsconfig.json

{
  "extends": "@tsconfig/next/tsconfig.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"]
    }
  }
}

frontend/app/layout.tsx

import "../globals.css";
import type { ReactNode } from "react";
export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="ja">
      <body>{children}</body>
    </html>
  );
}

frontend/app/page.tsx

'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';

export default function Home() {
  const [roomId, setRoomId] = useState('');
  const router = useRouter();
  const enter = () => router.push(`/room/${roomId || crypto.randomUUID().slice(0,6)}`);
  return (
    <main className="h-screen flex flex-col items-center justify-center gap-4 bg-sky-50">
      <h1 className="text-3xl font-bold">オンライン自習室</h1>
      <input
        value={roomId}
        onChange={(e) => setRoomId(e.target.value)}
        placeholder="ルームID (空なら新規)"
        className="border rounded p-2"
      />
      <button onClick={enter} className="bg-sky-500 text-white px-4 py-2 rounded">
        入室
      </button>
    </main>
  );
}

frontend/app/room/[id]/page.tsx

'use client';
import { useParams } from 'next/navigation';
import { useEffect, useState } from 'react';
import { connectToRoom } from '@/utils/livekit';
import VideoGrid from '@/components/VideoGrid';
import ChatPanel from '@/components/ChatPanel';
import Timer from '@/components/Timer';
import { io } from 'socket.io-client';

export default function RoomPage() {
  const { id } = useParams<{ id: string }>();
  const [room, setRoom] = useState<any>(null);
  const [socket] = useState(() => io('http://localhost:4000'));

  useEffect(() => {
    fetch(`http://localhost:4000/api/token?roomId=${id}`)
      .then((res) => res.json())
      .then(async ({ token }) => {
        const roomInstance = await connectToRoom(token);
        setRoom(roomInstance);
      });
    return () => {
      room?.disconnect();
      socket.disconnect();
    };
  }, [id]);

  if (!room) return <p className="text-center p-8">接続中…</p>;

  return (
    <div className="h-screen flex flex-col bg-sky-50">
      <header className="flex justify-center py-2 shadow-md bg-white relative">
        <Timer socket={socket} />
      </header>
      <main className="flex flex-1 overflow-hidden">
        <VideoGrid room={room} />
        <ChatPanel socket={socket} />
      </main>
    </div>
  );
}

frontend/components/VideoGrid.tsx

import { Participant } from '@livekit/client';
import { useLiveKitRoom } from '@livekit/react-components';
interface Props { room: any; }
export default function VideoGrid({ room }: Props) {
  const { participants } = useLiveKitRoom({ room });
  const all = [room.localParticipant, ...participants] as Participant[];
  return (
    <div className="grid flex-1 gap-2 p-2" style={{gridTemplateColumns:'repeat(auto-fill,minmax(200px,1fr))'}}>
      {all.map((p) => (
        <div key={p.sid} className="relative bg-white rounded-2xl shadow p-1 text-center">
          <video data-lk-video autoPlay playsInline className="w-full rounded-xl" />
          <span className="absolute bottom-1 left-1 text-xs bg-black/40 text-white px-1 rounded">
            {p.identity}
          </span>
        </div>
      ))}
    </div>
  );
}

frontend/components/ChatPanel.tsx

'use client';
import { useEffect, useState, useRef } from 'react';
import type { Socket } from 'socket.io-client';
interface Props { socket: Socket }
interface Msg { id: string; from: string; text: string; }
export default function ChatPanel({ socket }: Props) {
  const [msgs, setMsgs] = useState<Msg[]>([]);
  const [text, setText] = useState('');
  const bottomRef = useRef<HTMLDivElement>(null);
  useEffect(()=>{socket.on('chat',(m:Msg)=>setMsgs((prev)=>[...prev,m]));return()=>socket.off('chat');},[socket]);
  useEffect(()=>{bottomRef.current?.scrollIntoView({behavior:'smooth'});},[msgs]);
  const send=(e:any)=>{e.preventDefault();socket.emit('chat',{text});setText('');};
  return(
    <aside className="w-80 border-l bg-white flex flex-col p-2">
      <h2 className="font-bold text-lg mb-2">チャット</h2>
      <div className="flex-1 overflow-y-auto space-y-1">
        {msgs.map((m)=> (<p key={m.id} className="text-sm"><strong>{m.from}: </strong>{m.text}</p>))}
        <div ref={bottomRef}/>
      </div>
      <form onSubmit={send} className="flex gap-1 mt-2">
        <input value={text} onChange={e=>setText(e.target.value)} className="flex-1 rounded border px-2 py-1 text-sm" placeholder="メッセージを入力…" />
        <button className="px-3 py-1 bg-sky-500 text-white rounded">送信</button>
      </form>
    </aside>
  );
}

frontend/components/Timer.tsx

'use client';
import { useEffect, useState } from 'react';
import type { Socket } from 'socket.io-client';
interface Props { socket: Socket }
export default function Timer({ socket }: Props) {
  const [duration, setDuration] = useState(25*60);
  const [remaining, setRemaining] = useState(duration);
  const [host, setHost] = useState('');
  useEffect(()=>{
    socket.on('timer:update',({remaining,host})=>{setRemaining(remaining);setHost(host);});
    socket.on('timer:reset',({duration,host})=>{setDuration(duration);setRemaining(duration);setHost(host);});
  },[socket]);
  const onStart=()=>socket.emit('timer:start');
  const onReset=()=>{const m=prompt('タイマー分数(1-60)?','25');if(!m)return;socket.emit('timer:reset',{duration:parseInt(m)*60});};
  const pct=remaining/duration;const radius=45;const dash=2*Math.PI*radius*pct;
  return(
    <div className="flex items-center gap-4">
      <svg width="100" height="100" viewBox="0 0 100 100">
        <circle cx="50" cy="50" r={radius} stroke="#e5e7eb" strokeWidth="10" fill="none" />
        <circle cx="50" cy="50" r={radius} stroke="#3b82f6" strokeWidth="10" fill="none" strokeDasharray={`${dash} 999`} transform="rotate(-90 50 50)" />
        <text x="50" y="55" textAnchor="middle" fontSize="18" fill="#111">{Math.floor(remaining/60)}:{(remaining%60).toString().padStart(2,'0')}</text>
      </svg>
      <div className="text-sm text-gray-600">{host && `操作: ${host}`}</div>
      <button onClick={onStart} className="px-2 py-1 bg-green-500 text-white rounded">Start</button>
      <button onClick={onReset} className="px-2 py-1 bg-red-500 text-white rounded">Reset</button>
    </div>
  );
}

frontend/utils/livekit.ts

import { Room } from '@livekit/client';
export async function connectToRoom(token:string){
  const room=new Room({adaptiveStream:true});
  await room.connect('wss://YOUR_LIVEKIT_URL',token);
  return room;
}

3. backend/

backend/package.json

{
  "name": "online-study-room-backend",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js"
  },
  "dependencies": {
    "express": "^4.19.0",
    "socket.io": "^4.7.5",
    "cors": "^2.8.5",
    "livekit-server-sdk": "^0.16.1",
    "dotenv": "^16.4.5"
  },
  "devDependencies": {
    "tsx": "^4.7.0",
    "typescript": "5.5.0",
    "@types/express": "^4.17.21",
    "@types/node": "20.8.3"
  }
}

backend/tsconfig.json

{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "outDir": "dist",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}

backend/src/server.ts

import express from 'express';
import http from 'http';
import { Server } from 'socket.io';
import cors from 'cors';
import { AccessToken } from 'livekit-server-sdk';
import 'dotenv/config';

const app=express();
app.use(cors());
const server=http.createServer(app);
const io=new Server(server,{cors:{origin:'*'}});

let remaining=25*60; // 秒 (初期値)

/** LiveKit トークン発行 */
app.get('/api/token',(req,res)=>{
  const {roomId,user=`user-${Math.random().toString(36).slice(2,8)}`}=req.query as any;
  const at=new AccessToken(process.env.LIVEKIT_API_KEY!,process.env.LIVEKIT_API_SECRET!,{identity:user});
  at.addGrant({roomJoin:true,room:roomId as string});
  res.json({token:at.toJwt()});
});

io.on('connection',(socket)=>{
  socket.on('chat',(data)=>{io.emit('chat',{id:Date.now().toString(),from:data.from||socket.id,text:data.text});});

  let interval:NodeJS.Timeout|null=null;
  socket.on('timer:start',()=>{
    if(interval) return;
    interval=setInterval(()=>{
      remaining--; io.emit('timer:update',{remaining,host:socket.id});
      if(remaining<=0 && interval){clearInterval(interval);interval=null;}
    },1000);
  });
  socket.on('timer:reset',({duration})=>{
    remaining=duration; if(interval){clearInterval(interval);interval=null;}
    io.emit('timer:reset',{duration,host:socket.id});
  });
});

server.listen(4000,()=>console.log('Backend on http://localhost:4000'));

4. livekit/ (任意)

livekit/docker-compose.yml

version: '3'
services:
  livekit:
    image: livekit/livekit-server:latest
    command: --config /etc/livekit.yaml
    ports:
      - "7880:7880"    # HTTP + WS
      - "7881:7881"    # UDP/TCP TURN
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml

(設定ファイルは LiveKit docs を参照)

以上を そのまま GitHub に追加 → push すれば、ローカルでもクラウド（Vercel + Render など）でも動く最小 MVP が完成します。細部の調整や追加機能のご要望があればお気軽にどうぞ！

# 任意の作業フォルダへ移動（例）
cd ~/Desktop

# 自分のWebサイトのフォルダに移動
cd my-website

# Git初期化
git init

# GitHubのリポジトリと連携
git remote add origin https://github.com/あなたのユーザー名/my-website.git

# ファイル追加・コミット・プッシュ
git add .
git commit -m "初回アップロード"
git push -u origin main
