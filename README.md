// SnapishApp.jsx
// A single-file React + Tailwind example app that provides:
// - Simple sign-in (mocked, stored in localStorage)
// - Contacts list
// - Compose and send "texts" to multiple recipients via email addresses (calls a placeholder backend)
// - Chat UI inspired by Snapchat (bright, playful, ephemeral feeling) but not identical
//
// TODO: Replace the placeholder `sendEmail` fetch with your real backend/email provider (SendGrid, Mailgun, EmailJS, etc.).
// Backend should expose POST /api/send-email that accepts { from, to: [emails], subject, body }
// and handles sending emails securely. Never send email API keys from client-side.

import React, { useState, useEffect, useContext, createContext } from 'react';

/***********************
 * Simple Auth Context
 ***********************/
const AuthContext = createContext();
function AuthProvider({ children }) {
  const [user, setUser] = useState(() => {
    try {
      return JSON.parse(localStorage.getItem('snapish_user')) || null;
    } catch { return null; }
  });
  useEffect(() => {
    if (user) localStorage.setItem('snapish_user', JSON.stringify(user));
    else localStorage.removeItem('snapish_user');
  }, [user]);
  const signIn = (email, name) => setUser({ email, name });
  const signOut = () => setUser(null);
  return (
    <AuthContext.Provider value={{ user, signIn, signOut }}>
      {children}
    </AuthContext.Provider>
  );
}
function useAuth() { return useContext(AuthContext); }

/***********************
 * Mock contacts + persistence for messages
 ***********************/
const sampleContacts = [
  { id: 'c1', name: 'Ava', email: 'ava@example.com' },
  { id: 'c2', name: 'Liam', email: 'liam@example.com' },
  { id: 'c3', name: 'Noah', email: 'noah@example.com' },
];

function useMessages() {
  const [threads, setThreads] = useState(() => {
    try { return JSON.parse(localStorage.getItem('snapish_threads')) || {}; }
    catch { return {}; }
  });
  useEffect(() => { localStorage.setItem('snapish_threads', JSON.stringify(threads)); }, [threads]);
  const addMessage = (recipientsKey, message) => {
    setThreads(prev => {
      const copy = { ...prev };
      copy[recipientsKey] = (copy[recipientsKey] || []).concat(message);
      return copy;
    });
  };
  return { threads, addMessage };
}

/***********************
 * Helper: sendEmail (placeholder)
 ***********************/
async function sendEmail({ from, to, subject, body }) {
  // IMPORTANT: Do NOT put API keys here. Implement a secure server endpoint that accepts
  // email requests and communicates with an email provider.
  // Example: POST /api/send-email { from, to, subject, body } -> server sends email.
  // This client-side function demonstrates calling that endpoint.
  try {
    const res = await fetch('/api/send-email', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ from, to, subject, body }),
    });
    if (!res.ok) {
      const text = await res.text();
      throw new Error(text || 'Failed to send');
    }
    return await res.json();
  } catch (err) {
    console.warn('sendEmail error (placeholder):', err);
    // For developer flow, we'll still resolve as success so UI works in dev without backend.
    return { ok: true, debug: true };
  }
}

/***********************
 * Main App
 ***********************/
export default function SnapishApp() {
  return (
    <AuthProvider>
      <div className="min-h-screen bg-gradient-to-b from-yellow-50 to-white flex items-center justify-center p-4">
        <div className="w-full max-w-4xl bg-white rounded-2xl shadow-2xl overflow-hidden grid grid-cols-1 md:grid-cols-3">
          <Sidebar />
          <MainPanel />
        </div>
      </div>
    </AuthProvider>
  );
}

/***********************
 * Sidebar: Sign-in, Contacts
 ***********************/
function Sidebar() {
  const { user, signOut } = useAuth();
  return (
    <aside className="p-6 border-r md:col-span-1">
      <div className="flex items-center space-x-3 mb-6">
        <div className="h-12 w-12 rounded-full bg-yellow-400 flex items-center justify-center text-2xl font-bold text-white">S</div>
        <div>
          <div className="font-bold text-lg">Snapish</div>
          <div className="text-sm text-gray-500">Playful messaging</div>
        </div>
      </div>
      {user ? (
        <div>
          <div className="mb-4">
            <div className="text-sm text-gray-600">Signed in as</div>
            <div className="font-semibold">{user.name} <span className="text-xs text-gray-400">({user.email})</span></div>
          </div>
          <div className="mb-4">
            <CreateSnapButton />
          </div>
          <button onClick={signOut} className="w-full py-2 rounded-lg border text-sm">Sign out</button>
        </div>
      ) : (
        <SignInBox />
      )}

      <ContactsList />
    </aside>
  );
}

function SignInBox() {
  const { signIn } = useAuth();
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  return (
    <div className="space-y-2 mb-6">
      <input value={name} onChange={e=>setName(e.target.value)} placeholder="Your name" className="w-full p-2 border rounded-md" />
      <input value={email} onChange={e=>setEmail(e.target.value)} placeholder="you@example.com" className="w-full p-2 border rounded-md" />
      <button onClick={()=> signIn(email, name || email.split('@')[0])} className="w-full py-2 rounded-lg bg-yellow-400 text-white font-semibold">Sign in</button>
    </div>
  );
}

function ContactsList() {
  return (
    <div className="mt-6">
      <div className="text-xs text-gray-500 uppercase mb-2">Friends</div>
      <div className="space-y-2">
        {sampleContacts.map(c => (
          <div key={c.id} className="flex items-center justify-between p-2 rounded-xl hover:bg-gray-50">
            <div className="flex items-center space-x-3">
              <div className="h-10 w-10 rounded-full bg-gray-200 flex items-center justify-center">{c.name[0]}</div>
              <div>
                <div className="font-medium">{c.name}</div>
                <div className="text-xs text-gray-400">{c.email}</div>
              </div>
            </div>
            <div className="text-xs text-gray-400">Snap</div>
          </div>
        ))}
      </div>
    </div>
  );
}

/***********************
 * Main Panel: Chat list and active chat
 ***********************/
function MainPanel(){
  const { user } = useAuth();
  return (
    <main className="md:col-span-2 p-6">
      {!user ? (
        <div className="h-full flex flex-col items-center justify-center">
          <h2 className="text-2xl font-bold mb-2">Welcome to Snapish</h2>
          <p className="text-gray-500 mb-6 text-center">Sign in on the left to start sending playful email-based messages to your friends.</p>
        </div>
      ) : (
        <ChatApp />
      )}
    </main>
  );
}

function ChatApp(){
  const { threads, addMessage } = useMessages();
  const [activeThreadKey, setActiveThreadKey] = useState(null);
  const [filter, setFilter] = useState('');
  const threadKeys = Object.keys(threads).sort((a,b)=> (threads[b].slice(-1)[0]?.ts||0) - (threads[a].slice(-1)[0]?.ts||0));
  const filteredKeys = threadKeys.filter(k=> k.toLowerCase().includes(filter.toLowerCase()));
  useEffect(()=> { if (!activeThreadKey && filteredKeys.length) setActiveThreadKey(filteredKeys[0]); }, [filteredKeys]);

  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-6 h-[72vh]">
      <div className="col-span-1 border rounded-xl p-3 overflow-auto">
        <div className="flex items-center justify-between mb-3">
          <input placeholder="Search threads or emails" value={filter} onChange={e=>setFilter(e.target.value)} className="w-2/3 p-2 border rounded-md" />
          <NewThreadButton onCreate={(key)=> setActiveThreadKey(key)} />
        </div>

        <div className="space-y-2">
          {filteredKeys.length===0 && <div className="text-sm text-gray-400">No conversations yet.</div>}
          {filteredKeys.map(key => (
            <ThreadCard key={key} threadKey={key} last={threads[key].slice(-1)[0]} onOpen={()=>setActiveThreadKey(key)} selected={key===activeThreadKey} />
          ))}
        </div>
      </div>

      <div className="md:col-span-2 col-span-1 border rounded-xl p-4 flex flex-col">
        {activeThreadKey ? (
          <ThreadView threadKey={activeThreadKey} messages={threads[activeThreadKey]||[]} onSend={ (msg) => addMessage(activeThreadKey, msg) } />
        ) : (
          <div className="flex-1 flex items-center justify-center text-gray-400">Select or create a conversation to start</div>
        )}
      </div>
    </div>
  );
}

function ThreadCard({ threadKey, last, onOpen, selected }){
  return (
    <div onClick={onOpen} className={`p-2 rounded-lg flex items-center justify-between cursor-pointer ${selected ? 'bg-yellow-50' : 'hover:bg-gray-50'}`}>
      <div>
        <div className="font-semibold">{threadKey}</div>
        <div className="text-xs text-gray-500">{last ? last.preview : 'No messages yet'}</div>
      </div>
      <div className="text-xs text-gray-400">{last ? timeAgo(last.ts) : ''}</div>
    </div>
  );
}

function NewThreadButton({ onCreate }){
  const [open, setOpen] = useState(false);
  const [emails, setEmails] = useState('');
  const create = ()=>{
    const key = emails.split(',').map(s=>s.trim()).filter(Boolean).sort().join(', ');
    if (!key) return alert('Enter at least one email');
    setOpen(false);
    onCreate(key);
  };
  return (
    <div>
      <button onClick={()=>setOpen(v=>!v)} className="py-1 px-3 rounded-lg border">New</button>
      {open && (
        <div className="mt-2 bg-white p-3 rounded-lg shadow">
          <div className="text-xs text-gray-500 mb-1">Enter recipient emails, comma-separated</div>
          <input value={emails} onChange={e=>setEmails(e.target.value)} className="w-full p-2 border rounded-md" placeholder="friend@example.com, other@example.com" />
          <div className="flex justify-end mt-2 space-x-2">
            <button onClick={()=>setOpen(false)} className="py-1 px-3 rounded-lg border">Cancel</button>
            <button onClick={create} className="py-1 px-3 rounded-lg bg-yellow-400 text-white">Create</button>
          </div>
        </div>
      )}
    </div>
  );
}

function ThreadView({ threadKey, messages, onSend }){
  const { user } = useAuth();
  const [text, setText] = useState('');
  const [sending, setSending] = useState(false);

  const send = async () => {
    if (!text.trim()) return;
    setSending(true);
    const recipients = threadKey.split(',').map(s=>s.trim()).filter(Boolean);
    const msg = {
      id: Math.random().toString(36).slice(2),
      from: user.email,
      fromName: user.name,
      to: recipients,
      body: text,
      preview: text.slice(0,40) + (text.length>40? '...' : ''),
      ts: Date.now(),
    };
    // call backend (placeholder)
    await sendEmail({ from: `${user.name} <${user.email}>`, to: recipients, subject: 'Snapish message', body: text });
    onSend(msg);
    setText('');
    setSending(false);
  };

  return (
    <div className="flex flex-col h-full">
      <div className="border-b pb-3 mb-3">
        <div className="flex items-center justify-between">
          <div>
            <div className="font-bold">{threadKey}</div>
            <div className="text-xs text-gray-400">{messages.length} messages</div>
          </div>
          <div className="text-xs text-gray-400">{messages.slice(-1)[0]?.fromName || ''}</div>
        </div>
      </div>

      <div className="flex-1 overflow-auto space-y-3 pr-2">
        {messages.map(m => (
          <MessageBubble key={m.id} msg={m} me={m.from===user.email} />
        ))}
      </div>

      <div className="mt-3">
        <div className="flex space-x-2">
          <input value={text} onChange={e=>setText(e.target.value)} placeholder="Write a playful message..." className="flex-1 p-3 border rounded-2xl" />
          <button onClick={send} disabled={sending} className="px-4 rounded-2xl bg-yellow-400 text-white font-semibold">{sending? 'Sending...' : 'Snap'}</button>
        </div>
      </div>
    </div>
  );
}

function MessageBubble({ msg, me }){
  return (
    <div className={`max-w-[80%] ${me ? 'ml-auto text-right' : ''}`}>
      <div className={`inline-block p-3 rounded-2xl ${me ? 'bg-yellow-100' : 'bg-gray-100'}`}>
        <div className="text-sm">{msg.body}</div>
        <div className="text-xs text-gray-400 mt-1">{msg.fromName || msg.from} · {new Date(msg.ts).toLocaleTimeString()}</div>
      </div>
    </div>
  );
}

function CreateSnapButton(){
  const [open, setOpen] = useState(false);
  return (
    <div>
      <button onClick={()=>setOpen(v=>!v)} className="w-full py-2 rounded-lg bg-gradient-to-r from-yellow-400 to-yellow-500 text-white font-semibold">Create Snap</button>
      {open && <div className="text-xs text-gray-400 mt-2">Use the New button in the conversations panel to start a thread by entering emails.</div>}
    </div>
  );
}

/***********************
 * Utilities
 ***********************/
function timeAgo(ts){
  if (!ts) return '';
  const diff = Date.now() - ts;
  const sec = Math.floor(diff/1000);
  if (sec < 60) return `${sec}s`;
  const min = Math.floor(sec/60);
  if (min < 60) return `${min}m`;
  const hr = Math.floor(min/60);
  if (hr < 24) return `${hr}h`;
  const d = Math.floor(hr/24);
  return `${d}d`;
}

// End of file
