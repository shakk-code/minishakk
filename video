import React, { useState, useEffect, useRef, useCallback } from 'react';
import { 
  Play, Pause, SkipBack, SkipForward, Plus, Trash2, 
  Settings, Download, Save, Upload, Layers, Type, 
  Video, Music, Image as ImageIcon, Scissors, Monitor,
  ChevronRight, ChevronDown, Key, MousePointer2,
  FileVideo, MoreVertical, Home, X, Copy, Mic, Gauge, 
  Move, Wand2, Check, Menu
} from 'lucide-react';

// --- CONSTANTS & UTILS ---
const DB_NAME = 'WebCutProDB';
const DB_VERSION = 1;
const STORE_ASSETS = 'assets';

// IndexedDB for large asset storage
const initDB = () => {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open(DB_NAME, DB_VERSION);
    request.onerror = () => reject(request.error);
    request.onsuccess = () => resolve(request.result);
    request.onupgradeneeded = (event) => {
      const db = event.target.result;
      if (!db.objectStoreNames.contains(STORE_ASSETS)) {
        db.createObjectStore(STORE_ASSETS, { keyPath: 'id' });
      }
    };
  });
};

const saveAssetToDB = async (asset) => {
  const db = await initDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_ASSETS, 'readwrite');
    const store = tx.objectStore(STORE_ASSETS);
    store.put(asset);
    tx.oncomplete = () => resolve();
    tx.onerror = () => reject(tx.error);
  });
};

const loadAssetsFromDB = async () => {
  const db = await initDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_ASSETS, 'readonly');
    const store = tx.objectStore(STORE_ASSETS);
    const request = store.getAll();
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
};

const generateId = () => Math.random().toString(36).substr(2, 9);
const formatTime = (seconds) => {
  const m = Math.floor(seconds / 60);
  const s = Math.floor(seconds % 60);
  const ms = Math.floor((seconds % 1) * 100);
  return `${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}.${ms.toString().padStart(2, '0')}`;
};

const SUPPORTED_MIME_TYPES = [
  'video/webm;codecs=vp9',
  'video/webm;codecs=h264',
  'video/mp4',
].filter(type => MediaRecorder.isTypeSupported(type));

// --- COMPONENT: RENDER ENGINE ---
const RenderEngine = ({ 
  project, currentTime, playing, canvasRef, assets, fpsOverride 
}) => {
  const videoRefs = useRef({});
  const getAssetUrl = (assetId) => assets.find(a => a.id === assetId)?.url;

  const drawFrame = useCallback((ctx, width, height, time) => {
    // Clear
    ctx.fillStyle = '#000000';
    ctx.fillRect(0, 0, width, height);

    // Sorting: Track Index 0 is bottom layer. Higher index = on top.
    // We assume standard painter's algo: draw bottom tracks first.
    // Filter out muted or non-soloed tracks if a solo exists
    const soloTrackId = project.tracks.find(t => t.solo)?.id;
    
    project.tracks.forEach(track => {
      if (track.muted) return;
      if (soloTrackId && track.id !== soloTrackId) return;

      // Find clip
      const clip = track.clips.find(c => time >= c.start && time < c.start + c.duration);
      
      if (clip) {
        // Calculate clip-local time based on speed and offset
        // localTime = (globalTime - clipStart) * speed + offset
        const speed = clip.speed || 1;
        const clipTime = (time - clip.start) * speed + clip.offset;

        // Interpolation Helper
        const getProp = (propName, defaultValue) => {
          const kfs = clip.keyframes?.[propName];
          if (!kfs || kfs.length === 0) return defaultValue;
          const sorted = [...kfs].sort((a, b) => a.time - b.time);
          const relativeT = time - clip.start; // Keyframes relative to clip start
          
          if (relativeT <= sorted[0].time) return sorted[0].value;
          if (relativeT >= sorted[sorted.length - 1].time) return sorted[sorted.length - 1].value;
          
          const nextIdx = sorted.findIndex(k => k.time > relativeT);
          const prev = sorted[nextIdx - 1];
          const next = sorted[nextIdx];
          const ratio = (relativeT - prev.time) / (next.time - prev.time);
          return prev.value + (next.value - prev.value) * ratio;
        };

        // Properties
        const x = getProp('x', clip.x || width / 2);
        const y = getProp('y', clip.y || height / 2);
        const scale = getProp('scale', clip.scale || 1);
        const rotation = getProp('rotation', clip.rotation || 0);
        const opacity = getProp('opacity', clip.opacity !== undefined ? clip.opacity : 100) / 100;
        
        // Color Grading
        const brightness = getProp('brightness', 100);
        const contrast = getProp('contrast', 100);
        const saturate = getProp('saturate', 100);
        const hue = getProp('hue', 0);

        ctx.save();
        
        // Adjustment Layer Logic: applies to everything drawn BEFORE it
        if (clip.type === 'adjustment') {
            ctx.globalCompositeOperation = 'source-over'; // Actually complex to do real adjustment in 2D without buffer
            // Simulating adjustment by drawing a semi-transparent overlay with mix-blend-mode for visual effect
            // Real filter implementation requires grabbing ImageData, which is slow.
            // We will apply global filter to context for this specific rect, but it won't "filter" the background, just the new draw.
            // For this demo, Adjustment Layers will act as Overlay Filters (e.g., color tint)
            ctx.globalAlpha = opacity * 0.5;
            ctx.fillStyle = clip.color || 'orange';
            ctx.globalCompositeOperation = 'overlay';
            ctx.fillRect(0, 0, width, height);
            ctx.restore();
            return;
        }

        ctx.globalAlpha = opacity;
        ctx.filter = `brightness(${brightness}%) contrast(${contrast}%) saturate(${saturate}%) hue-rotate(${hue}deg)`;

        ctx.translate(x, y);
        ctx.rotate((rotation * Math.PI) / 180);
        ctx.scale(scale, scale);

        if (clip.type === 'video' || clip.type === 'image') {
          const url = getAssetUrl(clip.assetId);
          if (url) {
            let el = videoRefs.current[clip.assetId];
            if (!el) {
              if (clip.type === 'video') {
                el = document.createElement('video');
                el.src = url;
                el.muted = true;
                el.playsInline = true;
                el.crossOrigin = "anonymous";
              } else {
                el = new Image();
                el.src = url;
              }
              videoRefs.current[clip.assetId] = el;
            }

            if (clip.type === 'video') {
              if (el.readyState >= 2) {
                  // Sync video time
                  if (Math.abs(el.currentTime - clipTime) > 0.2) {
                    el.currentTime = clipTime;
                  }
              }
              ctx.drawImage(el, -el.videoWidth/2, -el.videoHeight/2);
            } else if (el.complete) {
              ctx.drawImage(el, -el.width/2, -el.height/2);
            }
          }
        } else if (clip.type === 'text') {
          ctx.font = `${clip.fontSize || 60}px ${clip.fontFamily || 'Arial'}`;
          ctx.fillStyle = clip.color || '#ffffff';
          ctx.textAlign = 'center';
          ctx.textBaseline = 'middle';
          ctx.fillText(clip.content || 'Text', 0, 0);
        }

        ctx.restore();
      }
    });
  }, [project.tracks, assets]);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    // Maintain internal resolution
    canvas.width = project.width;
    canvas.height = project.height;

    let handle;
    const render = () => {
      drawFrame(ctx, project.width, project.height, currentTime);
      if (playing) handle = requestAnimationFrame(render);
    };
    render();
    return () => cancelAnimationFrame(handle);
  }, [currentTime, playing, project, drawFrame]); // Re-bind when project changes

  return null;
};

// --- MAIN APPLICATION ---
export default function WebCutPro() {
  // Global View State
  const [view, setView] = useState('home'); // 'home' | 'editor'
  const [savedProjects, setSavedProjects] = useState([]);

  // Editor State
  const [project, setProject] = useState(null);
  const [assets, setAssets] = useState([]);
  const [currentTime, setCurrentTime] = useState(0);
  const [playing, setPlaying] = useState(false);
  
  // Tools & Interaction
  const [tool, setTool] = useState('select'); // 'select' | 'razor'
  const [scale, setScale] = useState(30); // px per second
  const [selectedClipId, setSelectedClipId] = useState(null);
  const [dragging, setDragging] = useState(null); // { type: 'clip'|'scrub', id, startX, originalStart, originalTrack }
  
  // UI State
  const [showSettings, setShowSettings] = useState(false);
  const [showExport, setShowExport] = useState(false);
  const [activeTab, setActiveTab] = useState('media'); // 'media' | 'effects'
  const [contextMenu, setContextMenu] = useState(null); // { x, y, type, data }
  
  // Preferences
  const [prefs, setPrefs] = useState({ previewFps: 30 });

  // Refs
  const canvasRef = useRef(null);
  const timelineRef = useRef(null);
  const playInterval = useRef(null);
  const timelineScrollRef = useRef(null);

  // --- LIFECYCLE: LOAD PROJECTS ---
  useEffect(() => {
    const load = () => {
      const stored = localStorage.getItem('webcut_projects');
      if (stored) setSavedProjects(JSON.parse(stored));
    };
    load();
    
    // IDB Assets
    loadAssetsFromDB().then(savedAssets => {
      setAssets(savedAssets.map(a => ({ ...a, url: URL.createObjectURL(a.blob) })));
    });
  }, []);

  // --- AUTOSAVE ---
  useEffect(() => {
    if (!project) return;
    const interval = setInterval(() => {
      const updatedList = savedProjects.map(p => p.id === project.id ? { ...project, lastModified: Date.now() } : p);
      if (!updatedList.find(p => p.id === project.id)) {
        updatedList.push({ ...project, lastModified: Date.now() });
      }
      setSavedProjects(updatedList);
      localStorage.setItem('webcut_projects', JSON.stringify(updatedList));
    }, 10000); // 10s
    return () => clearInterval(interval);
  }, [project, savedProjects]);

  // --- PLAYBACK LOOP ---
  useEffect(() => {
    if (playing) {
      const startTs = performance.now();
      const startOffset = currentTime;
      playInterval.current = requestAnimationFrame(function loop() {
        const now = performance.now();
        const dt = (now - startTs) / 1000;
        const nextT = startOffset + dt;
        if (nextT >= project.duration) {
          setPlaying(false);
          setCurrentTime(project.duration);
        } else {
          setCurrentTime(nextT);
          playInterval.current = requestAnimationFrame(loop);
        }
      });
    } else {
      cancelAnimationFrame(playInterval.current);
    }
    return () => cancelAnimationFrame(playInterval.current);
  }, [playing, project]); // Removed currentTime dep to prevent stutter

  // --- PROJECT MANAGEMENT HANDLERS ---
  const createProject = () => {
    const newProj = {
      id: generateId(),
      name: `Project ${new Date().toLocaleDateString()}`,
      width: 1920,
      height: 1080,
      fps: 30,
      duration: 120,
      lastModified: Date.now(),
      tracks: [
        { id: 'v1', type: 'video', name: 'V1', clips: [], muted: false, solo: false },
        { id: 'v2', type: 'video', name: 'V2', clips: [], muted: false, solo: false },
        { id: 'a1', type: 'audio', name: 'A1', clips: [], muted: false, solo: false },
        { id: 'a2', type: 'audio', name: 'A2', clips: [], muted: false, solo: false }
      ]
    };
    setProject(newProj);
    setView('editor');
  };

  const openProject = (p) => {
    setProject(p);
    setView('editor');
  };

  const deleteProject = (e, pid) => {
    e.stopPropagation();
    const updated = savedProjects.filter(p => p.id !== pid);
    setSavedProjects(updated);
    localStorage.setItem('webcut_projects', JSON.stringify(updated));
  };

  // --- EDITOR ACTIONS ---
  const handleImport = async (e) => {
    const files = Array.from(e.target.files);
    for (const file of files) {
      const type = file.type.startsWith('video') ? 'video' : file.type.startsWith('audio') ? 'audio' : 'image';
      const asset = { 
        id: generateId(), name: file.name, type, blob: file, 
        url: URL.createObjectURL(file), duration: 5 
      };
      
      if (type !== 'image') {
        const el = document.createElement(type);
        el.src = asset.url;
        await new Promise(r => { el.onloadedmetadata = () => { asset.duration = el.duration; r(); }; });
      }
      
      setAssets(prev => [...prev, asset]);
      saveAssetToDB({ id: asset.id, name: asset.name, type, blob: file, duration: asset.duration });
    }
  };

  // Drag & Drop from Bin to Timeline
  const handleDrop = (e, trackId) => {
    e.preventDefault();
    const assetId = e.dataTransfer.getData('assetId');
    const effectType = e.dataTransfer.getData('effectType');
    const rect = e.currentTarget.getBoundingClientRect();
    const dropTime = (e.clientX - rect.left + timelineScrollRef.current.scrollLeft) / scale;

    if (assetId) {
        const asset = assets.find(a => a.id === assetId);
        const newClip = {
          id: generateId(), assetId, type: asset.type, name: asset.name,
          start: Math.max(0, dropTime), duration: asset.duration, offset: 0,
          speed: 1, volume: 1,
          x: project.width/2, y: project.height/2, scale: 1, rotation: 0, opacity: 100
        };
        setProject(p => ({
          ...p, tracks: p.tracks.map(t => t.id === trackId ? { ...t, clips: [...t.clips, newClip] } : t)
        }));
    } else if (effectType === 'adjustment') {
         const newClip = {
            id: generateId(), type: 'adjustment', name: 'Adjustment Layer',
            start: Math.max(0, dropTime), duration: 5, offset: 0,
            opacity: 100, brightness: 100, contrast: 100, saturate: 100, hue: 0
         };
         setProject(p => ({
            ...p, tracks: p.tracks.map(t => t.id === trackId ? { ...t, clips: [...t.clips, newClip] } : t)
          }));
    }
  };

  // --- TIMELINE INTERACTION ---
  const handleTimelineMouseDown = (e) => {
      // Only scrubbing if clicking the ruler area
      const rect = timelineRef.current.getBoundingClientRect();
      // Check if clicked in ruler area (top 24px)
      const isRuler = (e.clientY - rect.top) < 24;
      
      if (isRuler) {
          setDragging({ type: 'scrub' });
          const x = e.clientX - rect.left + timelineScrollRef.current.scrollLeft - 100; // 100 is track header width
          setCurrentTime(Math.max(0, x / scale));
      }
  };

  const handleClipMouseDown = (e, clip, trackId) => {
      e.stopPropagation();
      if (tool === 'razor') {
          // Split Clip
          const rect = e.currentTarget.getBoundingClientRect();
          const clickX = e.clientX - rect.left;
          const splitLocalTime = clickX / scale; // visual duration into clip
          
          // Actual calculation
          // The clip starts at `clip.start`. The mouse is at `clip.start + splitLocalTime`.
          const globalSplitTime = clip.start + splitLocalTime;
          const newOffset = clip.offset + (splitLocalTime * (clip.speed || 1));
          
          const clip1 = { ...clip, duration: splitLocalTime };
          const clip2 = { 
              ...clip, id: generateId(), 
              start: globalSplitTime, 
              duration: clip.duration - splitLocalTime,
              offset: newOffset
          };

          setProject(p => ({
              ...p,
              tracks: p.tracks.map(t => t.id === trackId ? {
                  ...t,
                  clips: t.clips.flatMap(c => c.id === clip.id ? [clip1, clip2] : c)
              } : t)
          }));
          return;
      } 

      // Select Tool
      setSelectedClipId(clip.id);
      setDragging({ 
          type: 'clip', 
          id: clip.id, 
          startX: e.clientX, 
          originalStart: clip.start, 
          trackId: trackId 
      });
  };

  const handleMouseMove = (e) => {
      if (!dragging) return;

      if (dragging.type === 'scrub') {
          const rect = timelineRef.current.getBoundingClientRect();
          const x = e.clientX - rect.left + timelineScrollRef.current.scrollLeft - 100;
          setCurrentTime(Math.max(0, x / scale));
      } else if (dragging.type === 'clip') {
          const dx = (e.clientX - dragging.startX) / scale;
          const newStart = Math.max(0, dragging.originalStart + dx);
          
          // Visual update only (commit on up could be better but reacting live is nicer)
          // Also allow track switching? (Simplified to same track for stability in this file)
          setProject(p => ({
              ...p,
              tracks: p.tracks.map(t => t.id === dragging.trackId ? {
                  ...t,
                  clips: t.clips.map(c => c.id === dragging.id ? { ...c, start: newStart } : c)
              } : t)
          }));
      }
  };

  const handleMouseUp = () => {
      setDragging(null);
  };

  // --- CONTEXT MENU ---
  const handleContextMenu = (e, type, data) => {
      e.preventDefault();
      setContextMenu({ x: e.clientX, y: e.clientY, type, data });
  };

  const execContextAction = (action) => {
      if (!contextMenu) return;
      const { type, data } = contextMenu;
      
      if (type === 'clip') {
          if (action === 'delete') {
              setProject(p => ({ ...p, tracks: p.tracks.map(t => ({ ...t, clips: t.clips.filter(c => c.id !== data.id) })) }));
              setSelectedClipId(null);
          } else if (action === 'duplicate') {
               // Logic to copy/paste next to it
          } else if (action === 'reset_speed') {
              // set speed 1
          }
      } else if (type === 'track') {
          if (action === 'delete') {
              setProject(p => ({ ...p, tracks: p.tracks.filter(t => t.id !== data.id) }));
          } else if (action === 'add_video') {
              setProject(p => ({ ...p, tracks: [...p.tracks, { id: generateId(), type: 'video', name: 'New Video', clips: [] }] }));
          }
      }
      setContextMenu(null);
  };

  // --- EXPORT ---
  const handleExport = async (mimeType) => {
    setPlaying(false);
    setCurrentTime(0);
    const stream = canvasRef.current.captureStream(project.fps);
    
    // Note: Capturing Audio in this single-file setup without a complex WebAudio graph 
    // mixed into the stream is difficult. We will export video-only for this demo or 
    // assume the browser mixes the <video> elements if they played (which they don't, we draw them).
    // A full implementation requires a Tone.js or WebAudioContext graph connected to a DestNode.
    
    const recorder = new MediaRecorder(stream, { mimeType, videoBitsPerSecond: 8000000 });
    const chunks = [];
    recorder.ondataavailable = e => chunks.push(e.data);
    recorder.onstop = () => {
        const blob = new Blob(chunks, { type: mimeType });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `webcut_export.${mimeType.includes('mp4') ? 'mp4' : 'webm'}`;
        a.click();
        setShowExport(false);
    };
    recorder.start();

    // Render Loop for Export
    const dt = 1 / project.fps;
    for (let t = 0; t < project.duration; t += dt) {
        setCurrentTime(t);
        await new Promise(r => setTimeout(r, 20)); // Wait for draw
    }
    recorder.stop();
  };

  // --- HELPER COMPONENTS ---
  
  const PropertiesPanel = () => {
    if (!selectedClipId) return <div className="p-6 text-center text-gray-500 text-sm">Select a clip to edit</div>;
    
    const track = project.tracks.find(t => t.clips.some(c => c.id === selectedClipId));
    const clip = track?.clips.find(c => c.id === selectedClipId);
    if (!clip) return null;

    const update = (k, v) => {
        setProject(p => ({
            ...p, tracks: p.tracks.map(t => ({
                ...t, clips: t.clips.map(c => c.id === clip.id ? { ...c, [k]: v } : c)
            }))
        }));
    };

    return (
        <div className="p-4 overflow-y-auto h-full custom-scrollbar">
            <div className="mb-4 pb-2 border-b border-zinc-700">
                <h3 className="font-bold text-white truncate">{clip.name}</h3>
                <div className="text-xs text-blue-400 mt-1 font-mono">{clip.id}</div>
            </div>
            
            <div className="space-y-6">
                <div className="space-y-3">
                    <h4 className="text-xs font-bold text-zinc-500 uppercase">Transform</h4>
                    <div className="flex justify-between items-center text-sm">
                        <label>Scale</label>
                        <input type="range" min="0" max="3" step="0.1" value={clip.scale || 1} onChange={e=>update('scale', parseFloat(e.target.value))} className="w-32 accent-blue-500"/>
                    </div>
                    <div className="flex justify-between items-center text-sm">
                        <label>Rotation</label>
                        <input type="number" value={clip.rotation || 0} onChange={e=>update('rotation', parseFloat(e.target.value))} className="bg-zinc-800 w-16 text-right p-1 rounded border border-zinc-700"/>
                    </div>
                    <div className="flex justify-between items-center text-sm">
                        <label>Opacity</label>
                        <input type="range" min="0" max="100" value={clip.opacity ?? 100} onChange={e=>update('opacity', parseFloat(e.target.value))} className="w-32 accent-blue-500"/>
                    </div>
                </div>

                {(clip.type === 'video' || clip.type === 'image') && (
                    <div className="space-y-3">
                         <h4 className="text-xs font-bold text-zinc-500 uppercase">Color</h4>
                         {['Brightness', 'Contrast', 'Saturate'].map(f => (
                             <div key={f} className="flex justify-between items-center text-sm">
                                 <label>{f}</label>
                                 <input 
                                    type="range" min="0" max="200" 
                                    value={clip[f.toLowerCase()] || 100} 
                                    onChange={e=>update(f.toLowerCase(), parseFloat(e.target.value))} 
                                    className="w-32 accent-blue-500"
                                />
                             </div>
                         ))}
                    </div>
                )}
                
                {clip.type === 'text' && (
                    <div className="space-y-3">
                        <h4 className="text-xs font-bold text-zinc-500 uppercase">Typography</h4>
                        <textarea 
                            value={clip.content} 
                            onChange={e => update('content', e.target.value)}
                            className="w-full bg-zinc-800 text-white p-2 rounded border border-zinc-700 text-sm"
                        />
                        <select 
                            value={clip.fontFamily || 'Arial'}
                            onChange={e => update('fontFamily', e.target.value)}
                            className="w-full bg-zinc-800 text-white p-2 rounded border border-zinc-700 text-sm"
                        >
                            <option value="Arial">Arial</option>
                            <option value="Times New Roman">Times New Roman</option>
                            <option value="Courier New">Courier New</option>
                            <option value="Georgia">Georgia</option>
                            <option value="Verdana">Verdana</option>
                            <option value="Impact">Impact</option>
                        </select>
                        <div className="flex items-center gap-2">
                            <input type="color" value={clip.color || '#ffffff'} onChange={e=>update('color', e.target.value)} className="bg-transparent h-8 w-8 cursor-pointer"/>
                            <input type="number" value={clip.fontSize || 60} onChange={e=>update('fontSize', parseInt(e.target.value))} className="bg-zinc-800 w-16 p-1 border border-zinc-700 rounded"/>
                        </div>
                    </div>
                )}

                <div className="space-y-3">
                    <h4 className="text-xs font-bold text-zinc-500 uppercase">Clip Info</h4>
                    <div className="flex justify-between text-xs text-zinc-400">
                        <span>Start</span>
                        <span>{formatTime(clip.start)}</span>
                    </div>
                    <div className="flex justify-between text-xs text-zinc-400">
                        <span>Duration</span>
                        <span>{clip.duration.toFixed(2)}s</span>
                    </div>
                    <button onClick={() => { setSelectedClipId(null); updateClip(clip.id, { ...clip, start: 0 }) }} className="w-full py-1 bg-zinc-800 hover:bg-zinc-700 rounded text-xs mt-2">
                        Reset Position
                    </button>
                </div>
            </div>
        </div>
    );
  };

  // --- MAIN VIEW SWITCH ---
  if (view === 'home') {
    return (
      <div className="min-h-screen bg-zinc-950 text-white flex flex-col font-sans">
        <div className="p-8 max-w-5xl mx-auto w-full">
           <div className="flex items-center gap-3 mb-10">
               <div className="bg-blue-600 p-2 rounded-lg shadow-lg shadow-blue-900/20">
                   <Monitor size={32} className="text-white"/>
               </div>
               <h1 className="text-4xl font-bold tracking-tight">WebCut <span className="text-blue-500">Pro</span></h1>
           </div>

           <div className="grid grid-cols-3 gap-6 mb-10">
               <button onClick={createProject} className="bg-blue-600 hover:bg-blue-500 text-white p-6 rounded-xl text-left shadow-lg transition-all hover:scale-[1.02] group">
                   <div className="bg-blue-500/30 w-12 h-12 rounded-full flex items-center justify-center mb-4 group-hover:bg-blue-500">
                       <Plus size={24}/>
                   </div>
                   <h3 className="font-bold text-lg">New Project</h3>
                   <p className="text-blue-200 text-sm mt-1">Start from scratch</p>
               </button>
               <div className="col-span-2 bg-zinc-900/50 border border-zinc-800 rounded-xl p-6 flex items-center justify-center text-zinc-500">
                   <div className="text-center">
                       <Upload size={40} className="mx-auto mb-2 opacity-50"/>
                       <p>Drop project files here to import</p>
                   </div>
               </div>
           </div>

           <h2 className="text-xl font-semibold mb-4 flex items-center gap-2"><FileVideo size={20}/> Recent Projects</h2>
           <div className="bg-zinc-900 border border-zinc-800 rounded-xl overflow-hidden">
               {savedProjects.length === 0 ? (
                   <div className="p-10 text-center text-zinc-500">No recent projects found.</div>
               ) : (
                   savedProjects.map(p => (
                       <div key={p.id} className="flex items-center justify-between p-4 border-b border-zinc-800 last:border-0 hover:bg-zinc-800/50 transition-colors cursor-pointer" onClick={() => openProject(p)}>
                           <div className="flex items-center gap-4">
                               <div className="w-16 h-10 bg-zinc-950 rounded border border-zinc-800 flex items-center justify-center text-xs text-zinc-600">
                                   {p.width}p
                               </div>
                               <div>
                                   <h4 className="font-medium text-white">{p.name}</h4>
                                   <p className="text-xs text-zinc-500">Last edited: {new Date(p.lastModified).toLocaleString()}</p>
                               </div>
                           </div>
                           <div className="flex items-center gap-4">
                               <span className="text-xs text-zinc-500 bg-zinc-950 px-2 py-1 rounded">{formatTime(p.duration)}</span>
                               <button onClick={(e) => deleteProject(e, p.id)} className="p-2 hover:bg-red-500/20 hover:text-red-400 text-zinc-600 rounded-full"><Trash2 size={16}/></button>
                           </div>
                       </div>
                   ))
               )}
           </div>
        </div>
      </div>
    );
  }

  // --- EDITOR VIEW ---
  return (
    <div 
      className="flex flex-col h-screen bg-zinc-950 text-gray-200 font-sans overflow-hidden select-none"
      onMouseMove={handleMouseMove}
      onMouseUp={handleMouseUp}
      onClick={() => setContextMenu(null)} // Close context menu
    >
      {/* CONTEXT MENU POPUP */}
      {contextMenu && (
          <div 
            className="fixed bg-zinc-800 border border-zinc-700 shadow-2xl rounded-lg py-1 z-50 w-48"
            style={{ top: contextMenu.y, left: contextMenu.x }}
          >
              {contextMenu.type === 'clip' && (
                  <>
                    <button className="w-full text-left px-4 py-2 text-sm hover:bg-zinc-700 flex gap-2"><Scissors size={14}/> Cut</button>
                    <button className="w-full text-left px-4 py-2 text-sm hover:bg-zinc-700 flex gap-2"><Copy size={14}/> Copy</button>
                    <div className="h-px bg-zinc-700 my-1"></div>
                    <button onClick={() => execContextAction('delete')} className="w-full text-left px-4 py-2 text-sm hover:bg-zinc-700 text-red-400 flex gap-2"><Trash2 size={14}/> Delete</button>
                    <div className="h-px bg-zinc-700 my-1"></div>
                    <button className="w-full text-left px-4 py-2 text-sm hover:bg-zinc-700 flex gap-2"><Gauge size={14}/> Speed / Duration</button>
                    <button className="w-full text-left px-4 py-2 text-sm hover:bg-zinc-700 flex gap-2"><Mic size={14}/> Audio Gain</button>
                  </>
              )}
              {contextMenu.type === 'track' && (
                  <>
                    <button onClick={() => execContextAction('add_video')} className="w-full text-left px-4 py-2 text-sm hover:bg-zinc-700">Add Track</button>
                    <button onClick={() => execContextAction('delete')} className="w-full text-left px-4 py-2 text-sm hover:bg-zinc-700 text-red-400">Delete Track</button>
                  </>
              )}
          </div>
      )}

      {/* MENU BAR */}
      <div className="h-8 bg-zinc-900 border-b border-zinc-800 flex items-center px-2 text-xs select-none">
          <div className="flex gap-1 mr-4">
              {['File', 'Edit', 'View', 'Help'].map(m => (
                  <button key={m} className="px-3 py-1 hover:bg-zinc-800 rounded text-zinc-400 hover:text-white transition-colors">
                      {m}
                  </button>
              ))}
          </div>
          <div className="ml-auto flex gap-2">
              <span className="text-zinc-600">{project.width}x{project.height} @ {project.fps}fps</span>
          </div>
      </div>

      {/* MAIN HEADER */}
      <header className="h-14 bg-zinc-950 border-b border-zinc-800 flex items-center justify-between px-4 shrink-0">
        <div className="flex items-center gap-3">
          <button onClick={() => setView('home')} className="p-2 hover:bg-zinc-800 rounded-full text-zinc-400"><Home size={18}/></button>
          <div className="h-6 w-px bg-zinc-800"></div>
          <h1 className="font-bold text-lg text-white">WebCut <span className="text-blue-500">Pro</span></h1>
          <span className="text-xs text-zinc-600 ml-2">{project.name}</span>
        </div>

        <div className="flex items-center gap-3">
           <button onClick={() => setShowExport(true)} className="bg-blue-600 hover:bg-blue-500 text-white px-4 py-1.5 rounded text-sm font-medium flex items-center gap-2">
               <Download size={16}/> Export
           </button>
        </div>
      </header>

      {/* EXPORT MODAL */}
      {showExport && (
          <div className="fixed inset-0 bg-black/70 z-50 flex items-center justify-center">
              <div className="bg-zinc-900 border border-zinc-700 p-6 rounded-xl w-96">
                  <h2 className="text-xl font-bold mb-4">Export Video</h2>
                  <p className="text-zinc-400 text-sm mb-4">Select a format supported by your browser:</p>
                  <div className="space-y-2">
                      {SUPPORTED_MIME_TYPES.map(mime => (
                          <button 
                            key={mime} 
                            onClick={() => handleExport(mime)}
                            className="w-full p-3 bg-zinc-800 hover:bg-zinc-700 rounded text-left text-sm border border-zinc-700"
                          >
                              {mime}
                          </button>
                      ))}
                  </div>
                  <button onClick={() => setShowExport(false)} className="mt-4 w-full py-2 text-zinc-400 hover:text-white">Cancel</button>
              </div>
          </div>
      )}

      {/* WORKSPACE */}
      <div className="flex-1 flex overflow-hidden">
        {/* LEFT PANEL (Assets/Effects) */}
        <div className="w-72 bg-zinc-900 border-r border-zinc-800 flex flex-col shrink-0">
           <div className="flex border-b border-zinc-800">
               <button onClick={() => setActiveTab('media')} className={`flex-1 py-3 text-xs font-bold uppercase tracking-wider border-b-2 ${activeTab === 'media' ? 'border-blue-500 text-white' : 'border-transparent text-zinc-500'}`}>Project Media</button>
               <button onClick={() => setActiveTab('effects')} className={`flex-1 py-3 text-xs font-bold uppercase tracking-wider border-b-2 ${activeTab === 'effects' ? 'border-blue-500 text-white' : 'border-transparent text-zinc-500'}`}>Effects</button>
           </div>

           {activeTab === 'media' ? (
               <>
                <div className="p-2">
                    <label className="flex items-center justify-center gap-2 w-full p-2 bg-zinc-800 hover:bg-zinc-700 rounded border border-zinc-700 cursor-pointer text-sm text-zinc-300 transition-colors">
                        <Plus size={16} /> Import Media
                        <input type="file" multiple accept="video/*,image/*,audio/*" className="hidden" onChange={handleImport} />
                    </label>
                </div>
                <div className="flex-1 overflow-y-auto p-2 grid grid-cols-2 gap-2 content-start custom-scrollbar">
                    {assets.map(asset => (
                        <div 
                            key={asset.id}
                            draggable
                            onDragStart={(e) => { e.dataTransfer.setData('assetId', asset.id); }}
                            className="aspect-square bg-zinc-800 rounded border border-zinc-700 p-1 relative group cursor-grab hover:border-blue-500 transition-colors"
                        >
                            {asset.type === 'video' || asset.type === 'image' ? (
                                <img src={asset.url} className="w-full h-full object-cover rounded bg-black" />
                            ) : (
                                <div className="w-full h-full flex items-center justify-center text-zinc-600"><Music size={32}/></div>
                            )}
                            <div className="absolute bottom-0 left-0 right-0 bg-black/80 text-[10px] p-1 truncate text-zinc-300 font-mono">
                                {asset.name}
                            </div>
                        </div>
                    ))}
                     <div 
                        className="aspect-square border-2 border-dashed border-zinc-800 rounded flex flex-col items-center justify-center text-zinc-600 hover:border-zinc-600 hover:text-zinc-400 cursor-pointer"
                        onClick={() => {
                             const newClip = {
                                id: generateId(), type: 'text', name: 'Text Layer', content: 'Title',
                                start: currentTime, duration: 5, offset: 0, color: '#ffffff', fontSize: 80
                             };
                             // Find top video track or create one? Simplified: add to V1 if empty or new track
                             setProject(p => ({ ...p, tracks: p.tracks.map((t,i)=>i===0?{...t, clips:[...t.clips, newClip]}:t)}));
                        }}
                    >
                        <Type size={24} />
                        <span className="text-[10px] mt-2">Quick Text</span>
                     </div>
                </div>
               </>
           ) : (
               <div className="p-4 space-y-2">
                   <div 
                        draggable 
                        onDragStart={(e) => { e.dataTransfer.setData('effectType', 'adjustment'); }}
                        className="p-3 bg-zinc-800 rounded border border-zinc-700 flex items-center gap-3 cursor-grab hover:bg-zinc-700"
                    >
                       <Layers size={20} className="text-purple-400"/>
                       <div>
                           <div className="text-sm font-medium">Adjustment Layer</div>
                           <div className="text-xs text-zinc-500">Apply effects to tracks below</div>
                       </div>
                   </div>
                   {/* More effects placeholders */}
                   <div className="p-3 bg-zinc-800/50 rounded border border-zinc-800 flex items-center gap-3 opacity-50 cursor-not-allowed">
                       <Wand2 size={20}/>
                       <div>
                           <div className="text-sm">Blur (Coming Soon)</div>
                       </div>
                   </div>
               </div>
           )}
        </div>

        {/* CENTER (Preview) */}
        <div className="flex-1 bg-black flex flex-col relative">
             <div className="flex-1 flex items-center justify-center p-8 overflow-hidden">
                 <div className="relative shadow-2xl group" style={{ aspectRatio: `${project.width}/${project.height}`, maxHeight: '100%', maxWidth: '100%' }}>
                     <canvas ref={canvasRef} className="w-full h-full bg-black block"/>
                     <RenderEngine 
                        project={project} 
                        currentTime={currentTime} 
                        playing={playing} 
                        canvasRef={canvasRef} 
                        assets={assets}
                        fpsOverride={prefs.previewFps}
                     />
                 </div>
             </div>
             
             {/* Player Controls */}
             <div className="h-14 bg-zinc-900 border-t border-zinc-800 flex items-center justify-center gap-6">
                 <div className="font-mono text-blue-400 text-lg w-32 text-center">{formatTime(currentTime)}</div>
                 <div className="flex items-center gap-4">
                     <button onClick={() => setCurrentTime(0)} className="text-zinc-400 hover:text-white"><SkipBack size={20}/></button>
                     <button 
                        onClick={() => setPlaying(!playing)} 
                        className="w-10 h-10 bg-white text-black rounded-full flex items-center justify-center hover:scale-110 transition-transform shadow-lg shadow-white/10"
                     >
                        {playing ? <Pause size={20} fill="black"/> : <Play size={20} fill="black" className="ml-1"/>}
                     </button>
                     <button onClick={() => setCurrentTime(Math.min(project.duration, currentTime+5))} className="text-zinc-400 hover:text-white"><SkipForward size={20}/></button>
                 </div>
                 <div className="w-32"></div> {/* Spacer for balance */}
             </div>
        </div>

        {/* RIGHT (Properties) */}
        <div className="w-80 bg-zinc-900 border-l border-zinc-800 shrink-0 flex flex-col">
            <div className="p-3 border-b border-zinc-800 font-bold text-xs text-zinc-500 uppercase">Properties</div>
            <PropertiesPanel />
        </div>
      </div>

      {/* TIMELINE */}
      <div className="h-80 bg-zinc-950 border-t border-zinc-800 flex flex-col shrink-0 relative" ref={timelineRef}>
         {/* Timeline Toolbar */}
         <div className="h-10 bg-zinc-900 border-b border-zinc-800 flex items-center px-4 justify-between">
             <div className="flex gap-1 bg-zinc-950 p-1 rounded-lg border border-zinc-800">
                 <button 
                    onClick={() => setTool('select')} 
                    className={`p-1.5 rounded ${tool === 'select' ? 'bg-blue-600 text-white' : 'text-zinc-400 hover:text-white'}`}
                    title="Selection Tool (V)"
                 >
                     <MousePointer2 size={16}/>
                 </button>
                 <button 
                    onClick={() => setTool('razor')} 
                    className={`p-1.5 rounded ${tool === 'razor' ? 'bg-blue-600 text-white' : 'text-zinc-400 hover:text-white'}`}
                    title="Razor Tool (C)"
                 >
                     <Scissors size={16}/>
                 </button>
             </div>
             
             <div className="flex items-center gap-3">
                 <span className="text-xs text-zinc-500">Zoom</span>
                 <input 
                    type="range" min="10" max="100" value={scale} 
                    onChange={e => setScale(parseInt(e.target.value))}
                    className="w-32 h-1 bg-zinc-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
                 />
             </div>
         </div>

         {/* Timeline Content */}
         <div 
            className="flex-1 overflow-auto relative custom-scrollbar bg-zinc-950 select-none"
            ref={timelineScrollRef}
            onMouseDown={handleTimelineMouseDown} // Capture ruler clicks
         >
             <div className="min-w-full relative pb-10" style={{ width: `${project.duration * scale + 300}px` }}>
                 
                 {/* Ruler */}
                 <div className="h-8 border-b border-zinc-800 sticky top-0 z-20 bg-zinc-950/90 backdrop-blur-sm">
                     <div className="absolute left-[100px] right-0 top-0 bottom-0">
                         {Array.from({ length: Math.ceil(project.duration) }).map((_, i) => (
                             <div key={i} className="absolute top-0 bottom-0 border-l border-zinc-800 pl-1" style={{ left: `${i * scale}px` }}>
                                 <span className="text-[10px] text-zinc-600 font-mono">{i % 5 === 0 ? formatTime(i) : ''}</span>
                             </div>
                         ))}
                     </div>
                 </div>

                 {/* Tracks */}
                 <div className="pt-2">
                     {project.tracks.map(track => (
                         <div 
                            key={track.id} 
                            className="flex h-20 mb-1 group relative"
                            onContextMenu={(e) => handleContextMenu(e, 'track', track)}
                         >
                             {/* Track Header */}
                             <div className="w-[100px] bg-zinc-900 border-r border-zinc-800 shrink-0 flex flex-col justify-between p-2 z-10 sticky left-0 border-b border-zinc-900">
                                 <div className="flex items-center gap-2">
                                     <span className="text-xs font-bold text-zinc-400">{track.name}</span>
                                 </div>
                                 <div className="flex gap-1">
                                     <button 
                                        onClick={() => setProject(p => ({ ...p, tracks: p.tracks.map(t => t.id === track.id ? { ...t, muted: !t.muted } : t) }))}
                                        className={`w-5 h-5 rounded text-[10px] font-bold flex items-center justify-center border ${track.muted ? 'bg-blue-600 border-blue-500 text-white' : 'bg-zinc-800 border-zinc-700 text-zinc-500'}`}
                                     >M</button>
                                     <button 
                                        onClick={() => setProject(p => ({ ...p, tracks: p.tracks.map(t => t.id === track.id ? { ...t, solo: !t.solo } : t) }))}
                                        className={`w-5 h-5 rounded text-[10px] font-bold flex items-center justify-center border ${track.solo ? 'bg-yellow-600 border-yellow-500 text-white' : 'bg-zinc-800 border-zinc-700 text-zinc-500'}`}
                                     >S</button>
                                 </div>
                             </div>
                             
                             {/* Lane */}
                             <div 
                                className="relative flex-1 bg-zinc-900/30 border-b border-zinc-800/50"
                                onDragOver={e => e.preventDefault()}
                                onDrop={e => handleDrop(e, track.id)}
                             >
                                 {track.clips.map(clip => (
                                     <div
                                        key={clip.id}
                                        className={`absolute top-1 bottom-1 rounded-md overflow-hidden cursor-pointer text-[10px] px-2 flex flex-col justify-center
                                            border shadow-sm transition-all
                                            ${selectedClipId === clip.id ? 'ring-2 ring-white z-10' : 'opacity-90 hover:opacity-100'}
                                            ${clip.type === 'video' ? 'bg-blue-900/60 border-blue-700 text-blue-100' : 
                                              clip.type === 'audio' ? 'bg-emerald-900/60 border-emerald-700 text-emerald-100' : 
                                              clip.type === 'adjustment' ? 'bg-purple-900/60 border-purple-500 text-purple-100 dashed-border' :
                                              'bg-orange-900/60 border-orange-700 text-orange-100'}
                                        `}
                                        style={{ 
                                            left: `${clip.start * scale}px`, 
                                            width: `${clip.duration * scale}px` 
                                        }}
                                        onMouseDown={(e) => handleClipMouseDown(e, clip, track.id)}
                                        onContextMenu={(e) => handleContextMenu(e, 'clip', clip)}
                                     >
                                         <div className="truncate font-semibold drop-shadow-md">{clip.name}</div>
                                         {clip.type === 'video' && (
                                             <div className="flex overflow-hidden opacity-30 h-full absolute inset-0 -z-10">
                                                 {/* Fake Thumbnails visual would go here */}
                                             </div>
                                         )}
                                     </div>
                                 ))}
                             </div>
                         </div>
                     ))}
                 </div>

                 {/* Playhead */}
                 <div 
                    className="absolute top-0 bottom-0 w-px bg-red-500 z-30 pointer-events-none"
                    style={{ left: `${100 + currentTime * scale}px` }}
                 >
                     <div className="w-3 h-3 -ml-1.5 bg-red-500 rotate-45 transform -mt-1.5 shadow-sm"></div>
                 </div>

             </div>
         </div>
      </div>
      <style>{`
        .custom-scrollbar::-webkit-scrollbar { width: 6px; height: 6px; }
        .custom-scrollbar::-webkit-scrollbar-track { background: #09090b; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #27272a; border-radius: 3px; }
        .custom-scrollbar::-webkit-scrollbar-thumb:hover { background: #3f3f46; }
        .cursor-scissors { cursor: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="6" cy="6" r="3"/><circle cx="6" cy="18" r="3"/><line x1="20" y1="4" x2="8.12" y2="15.88"/><line x1="14.47" y1="14.48" x2="20" y2="20"/><line x1="8.12" y1="8.12" x2="12" y2="12"/></svg>') 12 12, auto; }
      `}</style>
      {/* Apply dynamic cursor */}
      {tool === 'razor' && <style>{`* { cursor: cell !important; }`}</style>}
    </div>
  );
}
