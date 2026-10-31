```react
import React, { useState, useEffect, useRef } from 'react';
import { 
  Bot, Mic, MicOff, Send, User, Sparkles, BookOpen, Briefcase, CheckSquare, 
  Wrench, HardDrive, Settings, ShieldCheck, HelpCircle, LogOut, Sun, Moon, 
  Volume2, VolumeX, RotateCcw, Copy, Trash2, Calendar, FileText, CheckCircle2, 
  Zap, Award, Compass, Play, Plus, ChevronRight, MessageSquare, Flame, Lightbulb, 
  Globe, Search, ArrowRight, ArrowLeft, RefreshCw, Smartphone, Layers, X
} from 'lucide-react';

// --- Default System Prompt for SONY AI ---
const SONY_SYSTEM_PROMPT = `
You are SONY, an intelligent, helpful, friendly, and professional personal AI partner.
Your core persona details:
- Default greeting: "Hello Boss! 👋 I'm SONY, your personal AI partner. How can I help you today?"
- You understand Hindi, English, and Hinglish fluent code-switching.
- Always respond in the language used by the user (Hindi, English, or Hinglish).
- You explain complex academic topics in super simple Hindi when requested.
- Maintain a warm, encouraging, respectful ("Boss"), and smart persona.
- Provide accurate, concise, clear, and well-structured markdown answers.
- Never fake actions; admit when you do not know something.
- For student queries, explain step-by-step from beginner to advanced.
`;

export default function App() {
  // Navigation & Core App State
  const [activeTab, setActiveTab] = useState('dashboard'); // dashboard, chat, voice, student, career, personal, tools, memory, settings, profile
  const [apiKey, setApiKey] = useState(process.env.GEMINI_API_KEY || '');
  const [userProfile, setUserProfile] = useState({
    name: 'Boss',
    role: 'Student (RBSE Class 12 Arts)',
    board: 'RBSE',
    classLevel: '12th Arts',
    preferredLang: 'Hinglish',
    targetExam: 'Board Exams 2026',
    savedSubjects: ['Political Science', 'History', 'Hindi', 'English']
  });

  // Splash Screen State
  const [showSplash, setShowSplash] = useState(true);

  // Chat Engine State
  const [chatMessages, setChatMessages] = useState([
    {
      id: 'welcome-msg',
      sender: 'sony',
      text: "Hello Boss! 👋\nI'm SONY, your personal AI partner.\nHow can I help you today?",
      timestamp: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
    }
  ]);
  const [inputPrompt, setInputPrompt] = useState('');
  const [isGenerating, setIsGenerating] = useState(false);
  const chatBottomRef = useRef(null);

  // Speech Recognition & TTS State
  const [isListening, setIsListening] = useState(false);
  const [isSpeaking, setIsSpeaking] = useState(false);
  const [voiceLang, setVoiceLang] = useState('hi-IN'); // hi-IN or en-US
  const recognitionRef = useRef(null);

  // Student Assistant Mode State
  const [selectedSubject, setSelectedSubject] = useState('Political Science');
  const [studentSubTab, setStudentSubTab] = useState('notes'); // notes, questions, quiz, flashcards, planner
  const [studentTopicQuery, setStudentTopicQuery] = useState('');
  const [generatedStudyContent, setGeneratedStudyContent] = useState('');
  const [isStudyLoading, setIsStudyLoading] = useState(false);

  // Personal Assistant / Tasks & Notes State
  const [tasks, setTasks] = useState([
    { id: '1', title: 'Revise Political Science Chapter 1', completed: false, category: 'Study', priority: 'High' },
    { id: '2', title: 'Complete History Practice Quiz', completed: true, category: 'Study', priority: 'Medium' },
    { id: '3', title: 'Draft English Resume for Internship', completed: false, category: 'Career', priority: 'Low' }
  ]);
  const [newTaskTitle, setNewTaskTitle] = useState('');
  const [notes, setNotes] = useState([
    { id: 'n1', title: 'Cold War Era Key Points', content: 'Superpower rivalry between USA and USSR post WWII. Non-Aligned Movement led by India.', date: 'Oct 24' },
    { id: 'n2', title: 'Grammar Tips: Active vs Passive', content: 'Subject performs action in active voice. Subject receives action in passive voice.', date: 'Oct 28' }
  ]);
  const [newNoteTitle, setNewNoteTitle] = useState('');
  const [newNoteContent, setNewNoteContent] = useState('');
  const [isAddingNote, setIsAddingNote] = useState(false);

  // Smart Tools State
  const [activeTool, setActiveTool] = useState('translator'); // translator, summarizer, grammar, calculator, ideas
  const [toolInput, setToolInput] = useState('');
  const [toolOutput, setToolOutput] = useState('');
  const [isToolLoading, setIsToolLoading] = useState(false);

  // Memory & Preferences System State
  const [memories, setMemories] = useState([
    { id: 'm1', key: 'Preferred Learning Style', value: 'Simple Hindi Bullet Points with Real-world Examples' },
    { id: 'm2', key: 'Goal', value: 'Score 90%+ in RBSE Class 12 Arts Board Exams' },
    { id: 'm3', key: 'Weak Subject', value: 'Political Science World Politics maps' }
  ]);
  const [newMemKey, setNewMemKey] = useState('');
  const [newMemVal, setNewMemVal] = useState('');

  // Career Mode State
  const [careerTab, setCareerTab] = useState('resume'); // resume, interview, email, roadmap
  const [resumeData, setResumeData] = useState({ name: 'Your Name', education: 'RBSE Class 12th', skills: 'Communication, Basic Computers', experience: 'School Projects' });
  const [careerOutput, setCareerOutput] = useState('');
  const [isCareerLoading, setIsCareerLoading] = useState(false);

  // Initialize Speech Recognition Setup
  useEffect(() => {
    const timer = setTimeout(() => setShowSplash(false), 2200);

    if ('SpeechRecognition' in window || 'webkitSpeechRecognition' in window) {
      const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
      recognitionRef.current = new SpeechRecognition();
      recognitionRef.current.continuous = false;
      recognitionRef.current.interimResults = false;

      recognitionRef.current.onresult = (event) => {
        const transcript = event.results[0][0].transcript;
        setInputPrompt(transcript);
        setIsListening(false);
      };

      recognitionRef.current.onerror = () => setIsListening(false);
      recognitionRef.current.onend = () => setIsListening(false);
    }

    return () => clearTimeout(timer);
  }, []);

  // Auto-scroll chat to bottom
  useEffect(() => {
    chatBottomRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [chatMessages, isGenerating]);

  // --- Real Gemini API Call Logic ---
  const callGeminiAPI = async (prompt, systemInstruction = SONY_SYSTEM_PROMPT, history = []) => {
    const effectiveKey = apiKey || ''; // Uses server fallback or dynamic key
    const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent?key=${effectiveKey}`;

    // Format chat contents
    const contents = [];
    if (history.length > 0) {
      history.forEach((msg) => {
        contents.push({
          role: msg.sender === 'user' ? 'user' : 'model',
          parts: [{ text: msg.text }]
        });
      });
    }
    contents.push({ role: 'user', parts: [{ text: prompt }] });

    const payload = {
      contents: contents,
      systemInstruction: {
        parts: [{ text: systemInstruction }]
      }
    };

    try {
      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      if (!response.ok) {
        throw new Error(`API Error ${response.status}: Failed to reach Gemini engine.`);
      }

      const data = await response.json();
      const reply = data.candidates?.[0]?.content?.parts?.[0]?.text;
      return reply || "Sorry Boss, I couldn't process that response right now. Please try again!";
    } catch (err) {
      console.error(err);
      return `⚠️ Engine Notice: ${err.message || 'Unable to connect to SONY AI Cloud'}. Please verify your network connection or API settings.`;
    }
  };

  // --- Handle Send Message ---
  const handleSendMessage = async (textToSend = inputPrompt) => {
    if (!textToSend.trim() || isGenerating) return;

    const userMsg = {
      id: Date.now().toString(),
      sender: 'user',
      text: textToSend,
      timestamp: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
    };

    setChatMessages((prev) => [...prev, userMsg]);
    setInputPrompt('');
    setIsGenerating(true);

    const historyForContext = chatMessages.slice(-6); // pass recent turns
    const aiResponseText = await callGeminiAPI(textToSend, SONY_SYSTEM_PROMPT, historyForContext);

    const sonyMsg = {
      id: (Date.now() + 1).toString(),
      sender: 'sony',
      text: aiResponseText,
      timestamp: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
    };

    setChatMessages((prev) => [...prev, sonyMsg]);
    setIsGenerating(false);

    // Auto speak response if voice enabled
    if (isSpeaking) {
      speakText(aiResponseText);
    }
  };

  // --- Voice Controls ---
  const toggleListening = () => {
    if (!recognitionRef.current) {
      alert("Speech recognition is not supported in this browser.");
      return;
    }
    if (isListening) {
      recognitionRef.current.stop();
      setIsListening(false);
    } else {
      recognitionRef.current.lang = voiceLang;
      recognitionRef.current.start();
      setIsListening(true);
    }
  };

  const speakText = (text) => {
    if (!('speechSynthesis' in window)) return;
    window.speechSynthesis.cancel();

    const cleanText = text.replace(/[*#_`]/g, ''); // strip markdown for speech
    const utterance = new SpeechSynthesisUtterance(cleanText);
    utterance.lang = voiceLang;

    utterance.onstart = () => setIsSpeaking(true);
    utterance.onend = () => setIsSpeaking(false);
    utterance.onerror = () => setIsSpeaking(false);

    window.speechSynthesis.speak(utterance);
  };

  const stopSpeaking = () => {
    if ('speechSynthesis' in window) {
      window.speechSynthesis.cancel();
      setIsSpeaking(false);
    }
  };

  // --- Student Assistant Generation ---
  const handleGenerateStudentContent = async (actionType) => {
    if (!studentTopicQuery.trim()) return;
    setIsStudyLoading(true);

    let prompt = '';
    if (actionType === 'notes') {
      prompt = `Generate simple, easy-to-understand Hindi/Hinglish study notes for RBSE Class 12 ${selectedSubject}. Topic: "${studentTopicQuery}". Include bullet points, key definitions, and key exam concepts.`;
    } else if (actionType === 'questions') {
      prompt = `Provide top 5 important board exam questions with detailed answers in Hindi/Hinglish for Class 12 ${selectedSubject}. Topic: "${studentTopicQuery}".`;
    } else if (actionType === 'quiz') {
      prompt = `Create a 5-question multiple choice revision quiz with answer keys and simple explanations in Hindi for Class 12 ${selectedSubject}. Topic: "${studentTopicQuery}".`;
    }

    const res = await callGeminiAPI(prompt, "You are SONY, expert RBSE Class 12 tutor specializing in Arts subjects (Political Science, History, Hindi, English).");
    setGeneratedStudyContent(res);
    setIsStudyLoading(false);
  };

  // --- Smart Tools Execution ---
  const handleRunSmartTool = async () => {
    if (!toolInput.trim()) return;
    setIsToolLoading(true);

    let prompt = '';
    if (activeTool === 'translator') {
      prompt = `Translate the following text into accurate English and natural Hindi/Hinglish:\n\n${toolInput}`;
    } else if (activeTool === 'summarizer') {
      prompt = `Summarize the following content into concise, clear executive bullet points:\n\n${toolInput}`;
    } else if (activeTool === 'grammar') {
      prompt = `Correct any spelling or grammar mistakes in the following text, explain the corrections, and provide a polished improved version:\n\n${toolInput}`;
    } else if (activeTool === 'ideas') {
      prompt = `Brainstorm 5 innovative, practical ideas or solutions based on this topic:\n\n${toolInput}`;
    }

    const res = await callGeminiAPI(prompt, "You are SONY, a high-precision smart productivity engine.");
    setToolOutput(res);
    setIsToolLoading(false);
  };

  // --- Career Tools Execution ---
  const handleRunCareerTool = async () => {
    setIsCareerLoading(true);
    let prompt = '';
    if (careerTab === 'resume') {
      prompt = `Create a sleek professional beginner resume layout in markdown format based on:
Name: ${resumeData.name}
Education: ${resumeData.education}
Skills: ${resumeData.skills}
Projects/Exp: ${resumeData.experience}`;
    } else if (careerTab === 'interview') {
      prompt = `Provide top 5 common job interview questions with high-scoring sample answers in Hindi and English for a beginner candidate.`;
    } else if (careerTab === 'email') {
      prompt = `Draft a formal professional email template for job application / leave request / inquiry suitable for beginners.`;
    }

    const res = await callGeminiAPI(prompt, "You are SONY, senior HR and career counselor assistant.");
    setCareerOutput(res);
    setIsCareerLoading(false);
  };

  // Task Actions
  const toggleTask = (id) => {
    setTasks(tasks.map(t => t.id === id ? { ...t, completed: !t.completed } : t));
  };

  const addTask = () => {
    if (!newTaskTitle.trim()) return;
    setTasks([...tasks, { id: Date.now().toString(), title: newTaskTitle, completed: false, category: 'General', priority: 'Medium' }]);
    setNewTaskTitle('');
  };

  const deleteTask = (id) => {
    setTasks(tasks.filter(t => t.id !== id));
  };

  // Note Actions
  const addNote = () => {
    if (!newNoteTitle.trim()) return;
    setNotes([...notes, { id: Date.now().toString(), title: newNoteTitle, content: newNoteContent, date: new Date().toLocaleDateString() }]);
    setNewNoteTitle('');
    setNewNoteContent('');
    setIsAddingNote(false);
  };

  // Memory Actions
  const addMemory = () => {
    if (!newMemKey.trim() || !newMemVal.trim()) return;
    setMemories([...memories, { id: Date.now().toString(), key: newMemKey, value: newMemVal }]);
    setNewMemKey('');
    setNewMemVal('');
  };

  const deleteMemory = (id) => {
    setMemories(memories.filter(m => m.id !== id));
  };

  // Copy helper
  const copyToClipboard = (text) => {
    navigator.clipboard.writeText(text);
    alert("Copied to clipboard!");
  };

  // ==================== UI RENDER ====================

  if (showSplash) {
    return (
      <div className="fixed inset-0 bg-[#050714] text-white flex flex-col items-center justify-center z-50 p-6 select-none">
        <div className="relative mb-6">
          <div className="w-28 h-28 rounded-full bg-gradient-to-tr from-cyan-500 via-blue-600 to-purple-600 p-1 animate-spin-slow shadow-[0_0_50px_rgba(6,182,212,0.6)]">
            <div className="w-full h-full bg-[#070b1e] rounded-full flex items-center justify-center">
              <Bot className="w-14 h-14 text-cyan-400 animate-pulse" />
            </div>
          </div>
          <div className="absolute -top-1 -right-1 w-6 h-6 bg-cyan-400 rounded-full animate-ping" />
        </div>
        <h1 className="text-4xl font-extrabold tracking-wider bg-clip-text text-transparent bg-gradient-to-r from-cyan-400 via-blue-400 to-purple-500">
          SONY AI
        </h1>
        <p className="text-xs font-semibold text-cyan-300 mt-2 tracking-widest uppercase">YOUR AI PARTNER</p>
        <p className="text-[10px] text-gray-400 mt-6 tracking-wide">LEARN • WORK • GROW • TOGETHER</p>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-[#050714] text-gray-100 font-sans flex flex-col max-w-md mx-auto relative border-x border-cyan-900/30 shadow-2xl overflow-hidden">
      
      {/* Dynamic Background Glows */}
      <div className="absolute top-0 left-1/4 w-72 h-72 bg-blue-600/10 rounded-full blur-3xl pointer-events-none" />
      <div className="absolute bottom-1/3 right-0 w-80 h-80 bg-purple-600/10 rounded-full blur-3xl pointer-events-none" />

      {/* --- TOP HEADER NAVIGATION --- */}
      <header className="sticky top-0 z-30 bg-[#070c23]/80 backdrop-blur-md border-b border-cyan-500/20 px-4 py-3 flex items-center justify-between">
        <div className="flex items-center space-x-3 cursor-pointer" onClick={() => setActiveTab('dashboard')}>
          <div className="relative">
            <div className="w-10 h-10 rounded-full bg-gradient-to-tr from-cyan-500 to-purple-600 p-0.5 shadow-[0_0_15px_rgba(6,182,212,0.4)]">
              <div className="w-full h-full bg-[#070b1e] rounded-full flex items-center justify-center">
                <Bot className="w-5 h-5 text-cyan-400" />
              </div>
            </div>
            {isSpeaking && (
              <span className="absolute -bottom-0.5 -right-0.5 flex h-3 w-3">
                <span className="animate-ping absolute inline-flex h-full w-full rounded-full bg-cyan-400 opacity-75"></span>
                <span className="relative inline-flex rounded-full h-3 w-3 bg-cyan-500"></span>
              </span>
            )}
          </div>
          <div>
            <div className="flex items-center space-x-1.5">
              <span className="font-extrabold text-base tracking-wider bg-clip-text text-transparent bg-gradient-to-r from-cyan-400 to-purple-400">
                SONY AI
              </span>
              <span className="bg-cyan-500/20 border border-cyan-400/40 text-cyan-300 text-[9px] px-1.5 py-0.5 rounded-full font-bold">
                PRO
              </span>
            </div>
            <p className="text-[10px] text-gray-400 tracking-tight">YOUR AI PARTNER</p>
          </div>
        </div>

        <div className="flex items-center space-x-2">
          {isSpeaking && (
            <button onClick={stopSpeaking} className="p-2 bg-red-500/20 text-red-400 rounded-full border border-red-500/30 animate-pulse">
              <VolumeX className="w-4 h-4" />
            </button>
          )}
          <button 
            onClick={() => setActiveTab('profile')}
            className="w-8 h-8 rounded-full bg-cyan-950/60 border border-cyan-500/30 flex items-center justify-center text-cyan-300 hover:border-cyan-400 transition"
          >
            <User className="w-4 h-4" />
          </button>
        </div>
      </header>

      {/* --- MAIN CONTENT AREA CONTAINER --- */}
      <main className="flex-1 overflow-y-auto pb-24 p-4 space-y-4">

        {/* 1. DASHBOARD TAB */}
        {activeTab === 'dashboard' && (
          <div className="space-y-5 animate-fade-in">
            {/* Banner Card */}
            <div className="relative overflow-hidden rounded-2xl bg-gradient-to-r from-cyan-950/80 via-blue-950/70 to-purple-950/80 border border-cyan-500/30 p-5 shadow-lg">
              <div className="relative z-10">
                <div className="flex items-center space-x-2 mb-2">
                  <Sparkles className="w-4 h-4 text-cyan-400 animate-spin-slow" />
                  <span className="text-xs font-semibold text-cyan-300 uppercase tracking-widest">Active Assistant</span>
                </div>
                <h2 className="text-2xl font-bold text-white leading-tight">
                  Hello <span className="text-cyan-400">{userProfile.name}</span>! 👋
                </h2>
                <p className="text-xs text-gray-300 mt-1">
                  Ready to study, organize tasks, or build your career today?
                </p>

                <div className="mt-4 flex space-x-2">
                  <button 
                    onClick={() => setActiveTab('chat')}
                    className="flex-1 bg-gradient-to-r from-cyan-500 to-blue-600 hover:from-cyan-400 hover:to-blue-500 text-black font-bold text-xs py-2.5 rounded-xl flex items-center justify-center space-x-2 shadow-[0_0_15px_rgba(6,182,212,0.4)] transition"
                  >
                    <MessageSquare className="w-4 h-4" />
                    <span>Chat with SONY</span>
                  </button>
                  <button 
                    onClick={() => setActiveTab('voice')}
                    className="bg-purple-950/80 border border-purple-500/40 text-purple-300 p-2.5 rounded-xl hover:bg-purple-900/60 transition"
                  >
                    <Mic className="w-4 h-4" />
                  </button>
                </div>
              </div>
              <div className="absolute -bottom-6 -right-6 w-32 h-32 bg-cyan-500/10 rounded-full blur-2xl" />
            </div>

            {/* Core Action Grid */}
            <div className="grid grid-cols-2 gap-3">
              <div 
                onClick={() => setActiveTab('student')}
                className="bg-[#0b122e]/90 border border-cyan-500/20 hover:border-cyan-400/50 p-4 rounded-2xl cursor-pointer transition flex flex-col justify-between group"
              >
                <div className="w-10 h-10 rounded-xl bg-blue-500/10 border border-blue-500/30 flex items-center justify-center text-blue-400 group-hover:scale-110 transition">
                  <BookOpen className="w-5 h-5" />
                </div>
                <div className="mt-4">
                  <h3 className="font-bold text-sm text-white flex items-center justify-between">
                    Student Companion <ChevronRight className="w-4 h-4 text-cyan-400" />
                  </h3>
                  <p className="text-[11px] text-gray-400 mt-1">RBSE Class 12 Arts, Notes & Quiz</p>
                </div>
              </div>

              <div 
                onClick={() => setActiveTab('personal')}
                className="bg-[#0b122e]/90 border border-purple-500/20 hover:border-purple-400/50 p-4 rounded-2xl cursor-pointer transition flex flex-col justify-between group"
              >
                <div className="w-10 h-10 rounded-xl bg-purple-500/10 border border-purple-500/30 flex items-center justify-center text-purple-400 group-hover:scale-110 transition">
                  <CheckSquare className="w-5 h-5" />
                </div>
                <div className="mt-4">
                  <h3 className="font-bold text-sm text-white flex items-center justify-between">
                    Tasks & Calendar <ChevronRight className="w-4 h-4 text-purple-400" />
                  </h3>
                  <p className="text-[11px] text-gray-400 mt-1">To-dos, Notes & Reminders</p>
                </div>
              </div>

              <div 
                onClick={() => setActiveTab('career')}
                className="bg-[#0b122e]/90 border border-emerald-500/20 hover:border-emerald-400/50 p-4 rounded-2xl cursor-pointer transition flex flex-col justify-between group"
              >
                <div className="w-10 h-10 rounded-xl bg-emerald-500/10 border border-emerald-500/30 flex items-center justify-center text-emerald-400 group-hover:scale-110 transition">
                  <Briefcase className="w-5 h-5" />
                </div>
                <div className="mt-4">
                  <h3 className="font-bold text-sm text-white flex items-center justify-between">
                    Career & Skills <ChevronRight className="w-4 h-4 text-emerald-400" />
                  </h3>
                  <p className="text-[11px] text-gray-400 mt-1">Resumes & Interviews</p>
                </div>
              </div>

              <div 
                onClick={() => setActiveTab('tools')}
                className="bg-[#0b122e]/90 border border-amber-500/20 hover:border-amber-400/50 p-4 rounded-2xl cursor-pointer transition flex flex-col justify-between group"
              >
                <div className="w-10 h-10 rounded-xl bg-amber-500/10 border border-amber-500/30 flex items-center justify-center text-amber-400 group-hover:scale-110 transition">
                  <Wrench className="w-5 h-5" />
                </div>
                <div className="mt-4">
                  <h3 className="font-bold text-sm text-white flex items-center justify-between">
                    Smart AI Tools <ChevronRight className="w-4 h-4 text-amber-400" />
                  </h3>
                  <p className="text-[11px] text-gray-400 mt-1">Translator, Summarizer</p>
                </div>
              </div>
            </div>

            {/* Quick Status / Today's Tasks */}
            <div className="bg-[#070c23]/90 border border-cyan-500/20 rounded-2xl p-4 space-y-3">
              <div className="flex items-center justify-between">
                <div className="flex items-center space-x-2">
                  <Calendar className="w-4 h-4 text-cyan-400" />
                  <h4 className="font-bold text-xs uppercase text-gray-200 tracking-wider">Today's Focus Tasks</h4>
                </div>
                <button onClick={() => setActiveTab('personal')} className="text-xs text-cyan-400 hover:underline">View All</button>
              </div>

              <div className="space-y-2">
                {tasks.slice(0, 3).map((task) => (
                  <div key={task.id} className="flex items-center justify-between p-2.5 rounded-xl bg-[#0d1538]/70 border border-cyan-900/30">
                    <div className="flex items-center space-x-2.5">
                      <input 
                        type="checkbox" 
                        checked={task.completed} 
                        onChange={() => toggleTask(task.id)}
                        className="rounded border-cyan-500/40 text-cyan-500 focus:ring-0 bg-black/40"
                      />
                      <span className={`text-xs ${task.completed ? 'line-through text-gray-500' : 'text-gray-200'}`}>
                        {task.title}
                      </span>
                    </div>
                    <span className="text-[10px] px-2 py-0.5 rounded-full bg-cyan-950 text-cyan-300 border border-cyan-800/40">
                      {task.priority}
                    </span>
                  </div>
                ))}
              </div>
            </div>
          </div>
        )}

        {/* 2. SONY AI CHAT TAB */}
        {activeTab === 'chat' && (
          <div className="flex flex-col h-[78vh]">
            <div className="flex-1 overflow-y-auto space-y-3 pr-1">
              {chatMessages.map((msg) => (
                <div key={msg.id} className={`flex ${msg.sender === 'user' ? 'justify-end' : 'justify-start'}`}>
                  <div className={`max-w-[85%] rounded-2xl p-3.5 text-xs leading-relaxed space-y-1 shadow-md ${
                    msg.sender === 'user' 
                      ? 'bg-gradient-to-r from-cyan-600 to-blue-600 text-white rounded-br-none' 
                      : 'bg-[#0a112d] border border-cyan-500/20 text-gray-100 rounded-bl-none'
                  }`}>
                    {msg.sender === 'sony' && (
                      <div className="flex items-center justify-between border-b border-cyan-500/20 pb-1.5 mb-1.5">
                        <span className="font-bold text-[10px] text-cyan-400 uppercase tracking-widest flex items-center space-x-1">
                          <Bot className="w-3 h-3 mr-1 inline" /> SONY AI
                        </span>
                        <div className="flex space-x-1">
                          <button onClick={() => speakText(msg.text)} className="hover:text-cyan-300 p-0.5"><Volume2 className="w-3 h-3" /></button>
                          <button onClick={() => copyToClipboard(msg.text)} className="hover:text-cyan-300 p-0.5"><Copy className="w-3 h-3" /></button>
                        </div>
                      </div>
                    )}
                    <div className="whitespace-pre-wrap">{msg.text}</div>
                    <div className="text-[9px] text-right text-gray-400 mt-1">{msg.timestamp}</div>
                  </div>
                </div>
              ))}
              {isGenerating && (
                <div className="flex justify-start">
                  <div className="bg-[#0a112d] border border-cyan-500/20 p-3 rounded-2xl rounded-bl-none text-xs flex items-center space-x-2 text-cyan-400">
                    <RefreshCw className="w-4 h-4 animate-spin" />
                    <span>SONY AI is thinking...</span>
                  </div>
                </div>
              )}
              <div ref={chatBottomRef} />
            </div>

            {/* Input Bar */}
            <div className="pt-2">
              <div className="flex items-center space-x-2 bg-[#0a112d] border border-cyan-500/30 rounded-2xl p-2 shadow-lg">
                <input 
                  type="text" 
                  value={inputPrompt} 
                  onChange={(e) => setInputPrompt(e.target.value)}
                  onKeyDown={(e) => e.key === 'Enter' && handleSendMessage()}
                  placeholder="Ask SONY anything in Hindi/English..."
                  className="flex-1 bg-transparent text-xs text-white placeholder-gray-500 outline-none px-2"
                />
                <button 
                  onClick={toggleListening} 
                  className={`p-2 rounded-xl transition ${isListening ? 'bg-red-500 text-white animate-pulse' : 'bg-cyan-950 text-cyan-400 hover:bg-cyan-900'}`}
                >
                  <Mic className="w-4 h-4" />
                </button>
                <button 
                  onClick={() => handleSendMessage()}
                  disabled={isGenerating || !inputPrompt.trim()}
                  className="bg-gradient-to-r from-cyan-500 to-blue-600 text-black p-2 rounded-xl font-bold hover:opacity-90 disabled:opacity-50 transition"
                >
                  <Send className="w-4 h-4" />
                </button>
              </div>
            </div>
          </div>
        )}

        {/* 3. VOICE ASSISTANT INTERFACE */}
        {activeTab === 'voice' && (
          <div className="flex flex-col items-center justify-center h-[75vh] space-y-8 text-center animate-fade-in">
            <div className="relative">
              <div className={`w-40 h-40 rounded-full border-2 border-cyan-500/30 flex items-center justify-center relative ${isListening || isSpeaking ? 'animate-pulse' : ''}`}>
                <div className="w-32 h-32 rounded-full bg-gradient-to-tr from-cyan-500/20 via-blue-600/20 to-purple-600/20 border border-cyan-400/40 flex items-center justify-center shadow-[0_0_40px_rgba(6,182,212,0.3)]">
                  <Bot className={`w-16 h-16 ${isSpeaking ? 'text-cyan-300 animate-bounce' : 'text-cyan-400'}`} />
                </div>
              </div>
            </div>

            <div>
              <h3 className="text-xl font-bold text-white">SONY Voice Assistant</h3>
              <p className="text-xs text-gray-400 mt-1">Speak commands like "Hey Sony" or tap the microphone</p>
            </div>

            <div className="flex items-center space-x-3 bg-[#0a112d] border border-cyan-500/30 px-3 py-1.5 rounded-full text-xs">
              <span className="text-gray-400">Language:</span>
              <button 
                onClick={() => setVoiceLang('hi-IN')}
                className={`px-2 py-0.5 rounded-full ${voiceLang === 'hi-IN' ? 'bg-cyan-500 text-black font-bold' : 'text-gray-300'}`}
              >
                Hindi / Hinglish
              </button>
              <button 
                onClick={() => setVoiceLang('en-US')}
                className={`px-2 py-0.5 rounded-full ${voiceLang === 'en-US' ? 'bg-cyan-500 text-black font-bold' : 'text-gray-300'}`}
              >
                English
              </button>
            </div>

            <div className="flex space-x-4">
              <button 
                onClick={toggleListening}
                className={`p-5 rounded-full shadow-lg transition flex items-center justify-center ${
                  isListening 
                    ? 'bg-red-500 text-white animate-ping' 
                    : 'bg-gradient-to-tr from-cyan-500 to-blue-600 text-black shadow-[0_0_25px_rgba(6,182,212,0.5)]'
                }`}
              >
                {isListening ? <MicOff className="w-8 h-8" /> : <Mic className="w-8 h-8" />}
              </button>
              {isSpeaking && (
                <button 
                  onClick={stopSpeaking}
                  className="p-5 rounded-full bg-red-950 border border-red-500/40 text-red-400 hover:bg-red-900 transition"
                >
                  <VolumeX className="w-8 h-8" />
                </button>
              )}
            </div>
          </div>
        )}

        {/* 4. STUDENT MODE (RBSE CLASS 12 ARTS FOCUS) */}
        {activeTab === 'student' && (
          <div className="space-y-4 animate-fade-in">
            <div className="bg-[#0a112d] border border-cyan-500/30 p-4 rounded-2xl flex items-center justify-between">
              <div>
                <span className="text-[10px] font-bold text-cyan-400 uppercase tracking-wider">Target Board</span>
                <h3 className="text-sm font-bold text-white">{userProfile.board} {userProfile.classLevel}</h3>
              </div>
              <select 
                value={selectedSubject} 
                onChange={(e) => setSelectedSubject(e.target.value)}
                className="bg-[#050714] border border-cyan-500/40 text-xs text-cyan-300 rounded-xl p-2 outline-none"
              >
                {userProfile.savedSubjects.map((sub) => (
                  <option key={sub} value={sub}>{sub}</option>
                ))}
              </select>
            </div>

            {/* Sub Tabs */}
            <div className="flex space-x-1 bg-[#0a112d] p-1 rounded-xl border border-cyan-500/20 text-xs">
              <button 
                onClick={() => setStudentSubTab('notes')}
                className={`flex-1 py-1.5 rounded-lg text-center transition ${studentSubTab === 'notes' ? 'bg-cyan-500 text-black font-bold' : 'text-gray-400'}`}
              >
                Notes
              </button>
              <button 
                onClick={() => setStudentSubTab('questions')}
                className={`flex-1 py-1.5 rounded-lg text-center transition ${studentSubTab === 'questions' ? 'bg-cyan-500 text-black font-bold' : 'text-gray-400'}`}
              >
                Important Qs
              </button>
              <button 
                onClick={() => setStudentSubTab('quiz')}
                className={`flex-1 py-1.5 rounded-lg text-center transition ${studentSubTab === 'quiz' ? 'bg-cyan-500 text-black font-bold' : 'text-gray-400'}`}
              >
                Quiz
              </button>
            </div>

            {/* Query Form */}
            <div className="space-y-2">
              <label className="text-xs text-gray-300">Enter Chapter or Topic Name:</label>
              <div className="flex space-x-2">
                <input 
                  type="text" 
                  value={studentTopicQuery}
                  onChange={(e) => setStudentTopicQuery(e.target.value)}
                  placeholder="e.g., Cold War Era, Constitution, Mughal Empire..."
                  className="flex-1 bg-[#0a112d] border border-cyan-500/30 rounded-xl p-2.5 text-xs text-white outline-none"
                />
                <button 
                  onClick={() => handleGenerateStudentContent(studentSubTab)}
                  disabled={isStudyLoading || !studentTopicQuery.trim()}
                  className="bg-cyan-500 text-black px-4 rounded-xl font-bold text-xs hover:bg-cyan-400 disabled:opacity-50 transition"
                >
                  {isStudyLoading ? 'Generating...' : 'Generate'}
                </button>
              </div>
            </div>

            {/* Output Display */}
            {generatedStudyContent && (
              <div className="bg-[#0a112d] border border-cyan-500/30 p-4 rounded-2xl space-y-2">
                <div className="flex items-center justify-between border-b border-cyan-500/20 pb-2">
                  <h4 className="font-bold text-xs text-cyan-300 uppercase">Study Material ({selectedSubject})</h4>
                  <button onClick={() => copyToClipboard(generatedStudyContent)} className="text-gray-400 hover:text-white">
                    <Copy className="w-4 h-4" />
                  </button>
                </div>
                <div className="text-xs leading-relaxed text-gray-200 whitespace-pre-wrap">
                  {generatedStudyContent}
                </div>
              </div>
            )}
          </div>
        )}

        {/* 5. PERSONAL ASSISTANT & TASKS */}
        {activeTab === 'personal' && (
          <div className="space-y-4 animate-fade-in">
            <div className="bg-[#0a112d] border border-cyan-500/30 p-4 rounded-2xl space-y-3">
              <h3 className="text-sm font-bold text-white flex items-center space-x-2">
                <CheckSquare className="w-4 h-4 text-cyan-400" />
                <span>Daily Task Manager</span>
              </h3>

              <div className="flex space-x-2">
                <input 
                  type="text" 
                  value={newTaskTitle}
                  onChange={(e) => setNewTaskTitle(e.target.value)}
                  placeholder="Add a new task..."
                  className="flex-1 bg-[#050714] border border-cyan-500/30 rounded-xl p-2 text-xs text-white outline-none"
                />
                <button onClick={addTask} className="bg-cyan-500 text-black p-2 rounded-xl font-bold">
                  <Plus className="w-4 h-4" />
                </button>
              </div>

              <div className="space-y-2 pt-2">
                {tasks.map((task) => (
                  <div key={task.id} className="flex items-center justify-between p-2.5 rounded-xl bg-[#050714] border border-cyan-900/30">
                    <div className="flex items-center space-x-2">
                      <input 
                        type="checkbox" 
                        checked={task.completed} 
                        onChange={() => toggleTask(task.id)}
                        className="rounded border-cyan-500/40 text-cyan-500 focus:ring-0 bg-black/40"
                      />
                      <span className={`text-xs ${task.completed ? 'line-through text-gray-500' : 'text-gray-200'}`}>
                        {task.title}
                      </span>
                    </div>
                    <button onClick={() => deleteTask(task.id)} className="text-red-400 hover:text-red-300">
                      <Trash2 className="w-3.5 h-3.5" />
                    </button>
                  </div>
                ))}
              </div>
            </div>

            {/* Quick Notes Tool */}
            <div className="bg-[#0a112d] border border-purple-500/30 p-4 rounded-2xl space-y-3">
              <div className="flex items-center justify-between">
                <h3 className="text-sm font-bold text-white flex items-center space-x-2">
                  <FileText className="w-4 h-4 text-purple-400" />
                  <span>Personal Notes</span>
                </h3>
                <button onClick={() => setIsAddingNote(!isAddingNote)} className="text-xs text-purple-400 flex items-center space-x-1">
                  <Plus className="w-3.5 h-3.5" /> <span>New Note</span>
                </button>
              </div>

              {isAddingNote && (
                <div className="space-y-2 p-3 bg-[#050714] rounded-xl border border-purple-500/20">
                  <input 
                    type="text" 
                    placeholder="Note Title" 
                    value={newNoteTitle} 
                    onChange={(e) => setNewNoteTitle(e.target.value)}
                    className="w-full bg-transparent border-b border-purple-500/30 p-1 text-xs text-white outline-none"
                  />
                  <textarea 
                    placeholder="Note Content..." 
                    value={newNoteContent} 
                    onChange={(e) => setNewNoteContent(e.target.value)}
                    className="w-full bg-transparent p-1 text-xs text-gray-300 outline-none h-16 resize-none"
                  />
                  <button onClick={addNote} className="w-full bg-purple-600 text-white font-bold text-xs py-1.5 rounded-lg">Save Note</button>
                </div>
              )}

              <div className="grid grid-cols-1 gap-2">
                {notes.map((note) => (
                  <div key={note.id} className="p-3 bg-[#050714] border border-purple-900/30 rounded-xl space-y-1">
                    <div className="flex justify-between items-center">
                      <h4 className="font-bold text-xs text-purple-300">{note.title}</h4>
                      <span className="text-[9px] text-gray-500">{note.date}</span>
                    </div>
                    <p className="text-[11px] text-gray-300">{note.content}</p>
                  </div>
                ))}
              </div>
            </div>
          </div>
        )}

        {/* 6. CAREER & SKILLS MODE */}
        {activeTab === 'career' && (
          <div className="space-y-4 animate-fade-in">
            <div className="flex space-x-1 bg-[#0a112d] p-1 rounded-xl border border-emerald-500/30 text-xs">
              <button 
                onClick={() => setCareerTab('resume')}
                className={`flex-1 py-1.5 rounded-lg transition ${careerTab === 'resume' ? 'bg-emerald-500 text-black font-bold' : 'text-gray-400'}`}
              >
                Resume Builder
              </button>
              <button 
                onClick={() => setCareerTab('interview')}
                className={`flex-1 py-1.5 rounded-lg transition ${careerTab === 'interview' ? 'bg-emerald-500 text-black font-bold' : 'text-gray-400'}`}
              >
                Interview Prep
              </button>
            </div>

            {careerTab === 'resume' && (
              <div className="bg-[#0a112d] border border-emerald-500/30 p-4 rounded-2xl space-y-3">
                <h4 className="font-bold text-xs text-emerald-400 uppercase tracking-wider">Beginner Resume Form</h4>
                <div className="space-y-2 text-xs">
                  <input 
                    type="text" 
                    placeholder="Full Name" 
                    value={resumeData.name} 
                    onChange={(e) => setResumeData({...resumeData, name: e.target.value})}
                    className="w-full bg-[#050714] border border-emerald-500/20 rounded-xl p-2 text-white outline-none"
                  />
                  <input 
                    type="text" 
                    placeholder="Education (e.g. RBSE 12th Arts)" 
                    value={resumeData.education} 
                    onChange={(e) => setResumeData({...resumeData, education: e.target.value})}
                    className="w-full bg-[#050714] border border-emerald-500/20 rounded-xl p-2 text-white outline-none"
                  />
                  <input 
                    type="text" 
                    placeholder="Key Skills" 
                    value={resumeData.skills} 
                    onChange={(e) => setResumeData({...resumeData, skills: e.target.value})}
                    className="w-full bg-[#050714] border border-emerald-500/20 rounded-xl p-2 text-white outline-none"
                  />
                </div>
                <button 
                  onClick={handleRunCareerTool}
                  disabled={isCareerLoading}
                  className="w-full bg-emerald-500 text-black font-bold text-xs py-2.5 rounded-xl hover:bg-emerald-400 transition"
                >
                  {isCareerLoading ? 'Building Resume...' : 'Generate Resume Format'}
                </button>
              </div>
            )}

            {careerOutput && (
              <div className="bg-[#0a112d] border border-emerald-500/30 p-4 rounded-2xl text-xs space-y-2">
                <div className="flex justify-between border-b border-emerald-500/20 pb-2">
                  <span className="font-bold text-emerald-400">Career Assistant Output</span>
                  <button onClick={() => copyToClipboard(careerOutput)} className="text-gray-400"><Copy className="w-4 h-4" /></button>
                </div>
                <div className="whitespace-pre-wrap text-gray-200">{careerOutput}</div>
              </div>
            )}
          </div>
        )}

        {/* 7. SMART AI TOOLS */}
        {activeTab === 'tools' && (
          <div className="space-y-4 animate-fade-in">
            {/* Tool Selection Grid */}
            <div className="grid grid-cols-4 gap-2">
              {[
                { id: 'translator', label: 'Translate', icon: Globe },
                { id: 'summarizer', label: 'Summarize', icon: FileText },
                { id: 'grammar', label: 'Grammar', icon: CheckCircle2 },
                { id: 'ideas', label: 'Ideas AI', icon: Lightbulb }
              ].map((tool) => {
                const Icon = tool.icon;
                return (
                  <button
                    key={tool.id}
                    onClick={() => setActiveTool(tool.id)}
                    className={`p-3 rounded-xl border flex flex-col items-center space-y-1 transition ${
                      activeTool === tool.id 
                        ? 'bg-amber-500/20 border-amber-400 text-amber-300 font-bold' 
                        : 'bg-[#0a112d] border-cyan-500/20 text-gray-400'
                    }`}
                  >
                    <Icon className="w-5 h-5" />
                    <span className="text-[10px]">{tool.label}</span>
                  </button>
                );
              })}
            </div>

            <div className="bg-[#0a112d] border border-amber-500/30 p-4 rounded-2xl space-y-3">
              <h4 className="font-bold text-xs text-amber-400 uppercase tracking-wider">
                Smart Tool: {activeTool.toUpperCase()}
              </h4>
              <textarea 
                value={toolInput}
                onChange={(e) => setToolInput(e.target.value)}
                placeholder="Enter text or prompt here..."
                className="w-full bg-[#050714] border border-amber-500/20 rounded-xl p-3 text-xs text-white outline-none h-24 resize-none"
              />
              <button 
                onClick={handleRunSmartTool}
                disabled={isToolLoading || !toolInput.trim()}
                className="w-full bg-gradient-to-r from-amber-500 to-orange-500 text-black font-bold text-xs py-2.5 rounded-xl hover:opacity-90 disabled:opacity-50 transition"
              >
                {isToolLoading ? 'Processing...' : 'Run Tool'}
              </button>
            </div>

            {toolOutput && (
              <div className="bg-[#0a112d] border border-amber-500/30 p-4 rounded-2xl text-xs space-y-2">
                <div className="flex justify-between border-b border-amber-500/20 pb-2">
                  <span className="font-bold text-amber-400">Tool Result</span>
                  <button onClick={() => copyToClipboard(toolOutput)} className="text-gray-400"><Copy className="w-4 h-4" /></button>
                </div>
                <div className="whitespace-pre-wrap text-gray-200">{toolOutput}</div>
              </div>
            )}
          </div>
        )}

        {/* 8. MEMORY & PREFERENCES */}
        {activeTab === 'memory' && (
          <div className="space-y-4 animate-fade-in">
            <div className="bg-[#0a112d] border border-cyan-500/30 p-4 rounded-2xl space-y-3">
              <h3 className="text-sm font-bold text-white flex items-center space-x-2">
                <HardDrive className="w-4 h-4 text-cyan-400" />
                <span>Permission-Based Memory System</span>
              </h3>
              <p className="text-xs text-gray-400">SONY remembers key details only with your explicit permission.</p>

              <div className="space-y-2 pt-2">
                <input 
                  type="text" 
                  placeholder="Memory Tag (e.g. Learning Style)" 
                  value={newMemKey} 
                  onChange={(e) => setNewMemKey(e.target.value)}
                  className="w-full bg-[#050714] border border-cyan-500/20 rounded-xl p-2 text-xs text-white outline-none"
                />
                <input 
                  type="text" 
                  placeholder="Value / Detail" 
                  value={newMemVal} 
                  onChange={(e) => setNewMemVal(e.target.value)}
                  className="w-full bg-[#050714] border border-cyan-500/20 rounded-xl p-2 text-xs text-white outline-none"
                />
                <button onClick={addMemory} className="w-full bg-cyan-500 text-black font-bold text-xs py-2 rounded-xl">Save Memory</button>
              </div>
            </div>

            <div className="space-y-2">
              {memories.map((mem) => (
                <div key={mem.id} className="p-3 bg-[#0a112d] border border-cyan-900/40 rounded-xl flex items-center justify-between">
                  <div>
                    <h5 className="font-bold text-xs text-cyan-400">{mem.key}</h5>
                    <p className="text-xs text-gray-300">{mem.value}</p>
                  </div>
                  <button onClick={() => deleteMemory(mem.id)} className="text-red-400 hover:text-red-300">
                    <Trash2 className="w-4 h-4" />
                  </button>
                </div>
              ))}
            </div>
          </div>
        )}

        {/* 9. USER PROFILE & SETTINGS */}
        {activeTab === 'profile' && (
          <div className="space-y-4 animate-fade-in">
            <div className="bg-[#0a112d] border border-cyan-500/30 p-4 rounded-2xl space-y-4">
              <div className="flex items-center space-x-3 border-b border-cyan-500/20 pb-4">
                <div className="w-14 h-14 rounded-full bg-gradient-to-tr from-cyan-500 to-purple-600 p-0.5">
                  <div className="w-full h-full bg-[#050714] rounded-full flex items-center justify-center text-cyan-300 font-bold text-lg">
                    {userProfile.name.charAt(0)}
                  </div>
                </div>
                <div>
                  <h3 className="font-bold text-white text-base">{userProfile.name}</h3>
                  <p className="text-xs text-cyan-400">{userProfile.role}</p>
                </div>
              </div>

              <div className="space-y-3 text-xs">
                <div>
                  <label className="text-gray-400">Gemini API Key (Optional Override):</label>
                  <input 
                    type="password" 
                    value={apiKey} 
                    onChange={(e) => setApiKey(e.target.value)}
                    placeholder="AI Studio Key..."
                    className="w-full bg-[#050714] border border-cyan-500/20 rounded-xl p-2 text-white outline-none mt-1"
                  />
                </div>

                <div>
                  <label className="text-gray-400">Preferred Language:</label>
                  <select 
                    value={userProfile.preferredLang} 
                    onChange={(e) => setUserProfile({...userProfile, preferredLang: e.target.value})}
                    className="w-full bg-[#050714] border border-cyan-500/20 rounded-xl p-2 text-white outline-none mt-1"
                  >
                    <option value="Hinglish">Hinglish (Hindi + English)</option>
                    <option value="Hindi">Hindi</option>
                    <option value="English">English</option>
                  </select>
                </div>
              </div>
            </div>

            <div className="bg-[#0a112d] border border-cyan-500/20 p-3 rounded-2xl space-y-2">
              <button onClick={() => setActiveTab('memory')} className="w-full flex items-center justify-between p-2 text-xs text-gray-200 hover:text-cyan-400">
                <span className="flex items-center space-x-2"><HardDrive className="w-4 h-4 text-cyan-400" /> <span>Memory Settings</span></span>
                <ChevronRight className="w-4 h-4" />
              </button>
              <button onClick={() => alert("SONY AI is in secure local mode. No private data shared.")} className="w-full flex items-center justify-between p-2 text-xs text-gray-200 hover:text-cyan-400">
                <span className="flex items-center space-x-2"><ShieldCheck className="w-4 h-4 text-emerald-400" /> <span>Privacy & Security</span></span>
                <ChevronRight className="w-4 h-4" />
              </button>
            </div>
          </div>
        )}
      </main>

      {/* --- BOTTOM FLOATING GLASS NAVIGATION BAR --- */}
      <nav className="fixed bottom-0 left-0 right-0 max-w-md mx-auto bg-[#070c23]/90 backdrop-blur-lg border-t border-cyan-500/30 px-3 py-2 flex items-center justify-around z-40">
        <button 
          onClick={() => setActiveTab('dashboard')}
          className={`flex flex-col items-center space-y-1 ${activeTab === 'dashboard' ? 'text-cyan-400' : 'text-gray-400'}`}
        >
          <Bot className="w-5 h-5" />
          <span className="text-[10px] font-medium">Home</span>
        </button>

        <button 
          onClick={() => setActiveTab('chat')}
          className={`flex flex-col items-center space-y-1 ${activeTab === 'chat' ? 'text-cyan-400' : 'text-gray-400'}`}
        >
          <MessageSquare className="w-5 h-5" />
          <span className="text-[10px] font-medium">Chat</span>
        </button>

        <button 
          onClick={() => setActiveTab('student')}
          className={`flex flex-col items-center space-y-1 ${activeTab === 'student' ? 'text-cyan-400' : 'text-gray-400'}`}
        >
          <BookOpen className="w-5 h-5" />
          <span className="text-[10px] font-medium">Study</span>
        </button>

        <button 
          onClick={() => setActiveTab('personal')}
          className={`flex flex-col items-center space-y-1 ${activeTab === 'personal' ? 'text-cyan-400' : 'text-gray-400'}`}
        >
          <CheckSquare className="w-5 h-5" />
          <span className="text-[10px] font-medium">Tasks</span>
        </button>

        <button 
          onClick={() => setActiveTab('tools')}
          className={`flex flex-col items-center space-y-1 ${activeTab === 'tools' ? 'text-cyan-400' : 'text-gray-400'}`}
        >
          <Wrench className="w-5 h-5" />
          <span className="text-[10px] font-medium">Tools</span>
        </button>
      </nav>
    </div>
  );
}
```
