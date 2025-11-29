import React, { useEffect, useMemo, useState } from "react";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Label } from "@/components/ui/label";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Checkbox } from "@/components/ui/checkbox";
import { Badge } from "@/components/ui/badge";
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table";
import { Separator } from "@/components/ui/separator";
import { CalendarDays, CheckCircle2, ClipboardList, Download, Upload, Trash2, BarChart3, School, Users, Search, Filter, PlusCircle, Sparkles } from "lucide-react";
import { ResponsiveContainer, BarChart, Bar, XAxis, YAxis, Tooltip } from "recharts";

// -----------------------------
// Types
// -----------------------------

type Homework = {
  id: string;
  title: string;
  subject: string;
  classLevel: string; // e.g., M.1/M.2/M.6
  assignedDate: string; // ISO string YYYY-MM-DD
  dueDate: string; // ISO
  description?: string;
  link?: string;
};

type StudentProgress = Record<string, Record<string, boolean>>; // studentName -> (homeworkId -> done)

// -----------------------------
// Constants
// -----------------------------

const SUBJECTS = [
  "Thai", "Mathematics", "Science", "Physics", "Chemistry", "Biology",
  "English", "Social Studies", "Computer", "Health & PE", "Art", "Music"
];

const CLASS_LEVELS = [
  "M.1", "M.2", "M.3", "M.4", "M.5", "M.6"
];

// Quick templates for faster input
const QUICK_TEMPLATES: Array<{ title: string; description?: string }> = [
  { title: "แบบฝึกหัดหน้า 45 ข้อ 1–10", description: "ให้แสดงวิธีทำทุกข้อ" },
  { title: "บทอ่าน Unit 3 + คำศัพท์ 20 คำ", description: "จดลงสมุดและถ่ายรูปแนบ" },
  { title: "Worksheet เรื่องแรงและการเคลื่อนที่", description: "ทำข้อ 1–8" },
];

// -----------------------------
// Local Storage Helpers
// -----------------------------

const LS_KEYS = {
  HOMEWORKS: "skhw_hw_items_v1",
  PROGRESS: "skhw_hw_progress_v1",
  SEEDED: "skhw_hw_seeded_v1",
};

function loadHomeworks(): Homework[] {
  try {
    // migration: read old keys if new data not found
    const rawNew = localStorage.getItem(LS_KEYS.HOMEWORKS);
    if (rawNew) return JSON.parse(rawNew) as Homework[];
    const rawOld = localStorage.getItem("skv_hw_items_v1");
    return rawOld ? (JSON.parse(rawOld) as Homework[]) : [];
  } catch {
    return [];
  }
}

function saveHomeworks(items: Homework[]) {
  localStorage.setItem(LS_KEYS.HOMEWORKS, JSON.stringify(items));
}

function loadProgress(): StudentProgress {
  try {
    const rawNew = localStorage.getItem(LS_KEYS.PROGRESS);
    if (rawNew) return JSON.parse(rawNew) as StudentProgress;
    const rawOld = localStorage.getItem("skv_hw_progress_v1");
    return rawOld ? (JSON.parse(rawOld) as StudentProgress) : {};
  } catch {
    return {};
  }
}

function saveProgress(p: StudentProgress) {
  localStorage.setItem(LS_KEYS.PROGRESS, JSON.stringify(p));
}

// -----------------------------
// Utilities
// -----------------------------

function isoToday() {
  const d = new Date();
  return d.toISOString().slice(0, 10);
}

function dueBadgeColor(daysLeft: number) {
  if (daysLeft < 0) return "destructive"; // overdue
  if (daysLeft === 0) return "secondary"; // today
  if (daysLeft <= 2) return "default"; // soon
  return "outline";
}

function daysUntil(dateStr: string) {
  const now = new Date();
  const due = new Date(dateStr + "T00:00:00");
  const diff = Math.ceil((due.getTime() - (now as any).getTime()) / (1000 * 60 * 60 * 24));
  return diff;
}

function uid() {
  return Math.random().toString(36).slice(2, 10) + Date.now().toString(36).slice(-4);
}

// nice date display
function displayDate(d: string) {
  try { return new Date(d + "T00:00:00").toLocaleDateString(); } catch { return d; }
}

// -----------------------------
// Main App
// -----------------------------

export default function HomeworkTrackerApp() {
  const [tab, setTab] = useState<string>("teacher");
  const [items, setItems] = useState<Homework[]>([]);
  const [progress, setProgress] = useState<StudentProgress>({});

  // UI state – teacher list filters
  const [tSearch, setTSearch] = useState("");
  const [tSubject, setTSubject] = useState<string | "ALL">("ALL");
  const [tClass, setTClass] = useState<string | "ALL">("ALL");

  useEffect(() => {
    const seeded = localStorage.getItem(LS_KEYS.SEEDED);
    const loaded = loadHomeworks();
    // Seed demo data for better first-time UI
    if (!seeded && loaded.length === 0) {
      const demo: Homework[] = [
        { id: uid(), title: "Worksheet บทที่ 2", subject: "Mathematics", classLevel: "M.2", assignedDate: isoToday(), dueDate: isoToday(), description: "ทำข้อ 1–10" },
        { id: uid(), title: "Lab Report เคมี", subject: "Chemistry", classLevel: "M.5", assignedDate: isoToday(), dueDate: isoToday(), description: "ใส่รูปผลทดลอง" },
        { id: uid(), title: "อ่านบทที่ 4 + สรุปย่อ", subject: "English", classLevel: "M.3", assignedDate: isoToday(), dueDate: isoToday(), description: "ครึ่งหน้า" },
      ];
      setItems(demo);
      localStorage.setItem(LS_KEYS.SEEDED, "1");
      saveHomeworks(demo);
    } else {
      setItems(loaded);
    }
    setProgress(loadProgress());
  }, []);

  useEffect(() => { saveHomeworks(items); }, [items]);
  useEffect(() => { saveProgress(progress); }, [progress]);

  // Teacher form state
  const [title, setTitle] = useState("");
  const [subject, setSubject] = useState<string>(SUBJECTS[0]);
  const [classLevel, setClassLevel] = useState<string>(CLASS_LEVELS[0]);
  const [assignedDate, setAssignedDate] = useState<string>(isoToday());
  const [dueDate, setDueDate] = useState<string>(isoToday());
  const [description, setDescription] = useState("");
  const [link, setLink] = useState("");

  // Student state
  const [studentName, setStudentName] = useState<string>("");
  const [studentClass, setStudentClass] = useState<string>(CLASS_LEVELS[0]);
  const [showCompleted, setShowCompleted] = useState<boolean>(true);

  // Filters & derived data
  const studentData = useMemo(() => progress[studentName] || {}, [progress, studentName]);

  const teacherList = useMemo(() => {
    return items
      .filter(i => (tSubject === "ALL" ? true : i.subject === tSubject))
      .filter(i => (tClass === "ALL" ? true : i.classLevel === tClass))
      .filter(i => {
        const q = tSearch.trim().toLowerCase();
        if (!q) return true;
        return (
          i.title.toLowerCase().includes(q) ||
          i.description?.toLowerCase().includes(q) ||
          i.subject.toLowerCase().includes(q) ||
          i.classLevel.toLowerCase().includes(q)
        );
      })
      .sort((a, b) => a.dueDate.localeCompare(b.dueDate));
  }, [items, tSubject, tClass, tSearch]);

  const filteredItems = useMemo(() => items
    .filter(i => i.classLevel === studentClass)
    .sort((a, b) => a.dueDate.localeCompare(b.dueDate)), [items, studentClass]);

  const visibleItems = useMemo(() => filteredItems.filter(i => {
    const done = !!studentData[i.id];
    return showCompleted ? true : !done;
  }), [filteredItems, studentData, showCompleted]);

  // Admin charts
  const chartBySubject = useMemo(() => {
    const map: Record<string, number> = {};
    items.forEach(i => { map[i.subject] = (map[i.subject] || 0) + 1; });
    return Object.entries(map).map(([name, total]) => ({ name, total }));
  }, [items]);

  const completionByClass = useMemo(() => {
    const classStats: Record<string, { done: number; total: number } > = {};
    for (const i of items) {
      if (!classStats[i.classLevel]) classStats[i.classLevel] = { done: 0, total: 0 };
      classStats[i.classLevel].total += 1;
    }
    // simple estimate: average completion across known students
    const students = Object.keys(progress);
    for (const stu of students) {
      const rec = progress[stu];
      for (const [hwId, done] of Object.entries(rec)) {
        if (!done) continue;
        const hw = items.find(x => x.id === hwId);
        if (hw) classStats[hw.classLevel].done += 1 / Math.max(students.length, 1);
      }
    }
    return Object.entries(classStats).map(([name, st]) => ({ name, percent: st.total ? Math.round((st.done / st.total) * 100) : 0 }));
  }, [items, progress]);

  // Handlers
  function addHomework() {
    if (!title.trim()) return;
    const hw: Homework = {
      id: uid(), title: title.trim(), subject, classLevel, assignedDate, dueDate, description, link
    };
    setItems(prev => [hw, ...prev]);
    // reset minimal
    setTitle("");
    setDescription("");
    setLink("");
  }

  function removeHomework(id: string) {
    setItems(prev => prev.filter(i => i.id !== id));
    setProgress(prev => {
      const clone: StudentProgress = { ...prev };
      for (const s of Object.keys(clone)) {
        if (clone[s] && clone[s][id] !== undefined) {
          const { [id]: _omit, ...rest } = clone[s];
          clone[s] = rest;
        }
      }
      return clone;
    });
  }

  function toggleDone(student: string, hwId: string, value: boolean) {
    setProgress(prev => {
      const srec = prev[student] || {};
      const next = { ...prev, [student]: { ...srec, [hwId]: value } };
      return next;
    });
  }

  function exportAll() {
    const blob = new Blob([JSON.stringify({ items, progress }, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `skhw-homework-${new Date().toISOString().slice(0,10)}.json`;
    a.click();
    URL.revokeObjectURL(url);
  }

  function onImportFile(e: React.ChangeEvent<HTMLInputElement>) {
    const f = e.target.files?.[0];
    if (!f) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const parsed = JSON.parse(String(reader.result));
        if (parsed.items && Array.isArray(parsed.items)) setItems(parsed.items);
        if (parsed.progress && typeof parsed.progress === "object") setProgress(parsed.progress);
      } catch {
        alert("Invalid JSON file");
      }
    };
    reader.readAsText(f);
  }

  // -----------------------------
  // UI subcomponents
  // -----------------------------

  function FilterChips() {
    return (
      <div className="flex flex-wrap items-center gap-2">
        <Badge variant="outline" className="px-3 py-1 rounded-full">ตัวกรอง</Badge>
        <Select value={tSubject} onValueChange={(v)=>setTSubject(v as any)}>
          <SelectTrigger className="w-44 h-9"><Filter className="size-4 mr-2"/><SelectValue placeholder="ทุกวิชา" /></SelectTrigger>
          <SelectContent>
            <SelectItem value="ALL">ทุกวิชา</SelectItem>
            {SUBJECTS.map(s => <SelectItem key={s} value={s}>{s}</SelectItem>)}
          </SelectContent>
        </Select>
        <Select value={tClass} onValueChange={(v)=>setTClass(v as any)}>
          <SelectTrigger className="w-36 h-9"><SelectValue placeholder="ทุกชั้น" /></SelectTrigger>
          <SelectContent>
            <SelectItem value="ALL">ทุกชั้น</SelectItem>
            {CLASS_LEVELS.map(c => <SelectItem key={c} value={c}>{c}</SelectItem>)}
          </SelectContent>
        </Select>
        <div className="flex-1 min-w-[180px]">
          <div className="relative">
            <Search className="size-4 absolute left-3 top-1/2 -translate-y-1/2 text-slate-400"/>
            <Input value={tSearch} onChange={(e)=>setTSearch(e.target.value)} placeholder="ค้นหาการบ้าน/คำอธิบาย/วิชา" className="pl-9"/>
          </div>
        </div>
      </div>
    );
  }

  function EmptyState() {
    return (
      <div className="flex flex-col items-center justify-center text-center py-10 text-slate-500">
        <Sparkles className="size-8 mb-2"/>
        <p>ยังไม่มีการบ้าน — เริ่มมอบหมายงานแรกของคุณเลย</p>
        <p className="text-xs">Tip: ใช้ปุ่ม "แม่แบบด่วน" เพื่อกรอกหัวข้ออย่างรวดเร็ว</p>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-b from-rose-50 via-slate-50 to-slate-100 text-slate-900">
      <header className="sticky top-0 z-20 backdrop-blur supports-[backdrop-filter]:bg-white/60 bg-white/70 border-b">
        <div className="max-w-6xl mx-auto px-4 py-3 flex items-center justify-between">
          <div className="flex items-center gap-3">
            <div className="size-10 rounded-2xl bg-rose-100 flex items-center justify-center shadow-sm">
              <School className="size-5" />
            </div>
            <div>
              <h1 className="text-xl font-bold tracking-tight">SKHW Homework Tracker</h1>
              <p className="text-xs text-slate-500">Smart Learning • Suankularb Wittayalai</p>
            </div>
          </div>
          <div className="flex items-center gap-2">
            <Button variant="outline" size="sm" onClick={exportAll} className="gap-2"><Download className="size-4"/>Export</Button>
            <label className="relative">
              <input type="file" accept="application/json" onChange={onImportFile} className="absolute inset-0 opacity-0 cursor-pointer"/>
              <Button variant="outline" size="sm" className="gap-2"><Upload className="size-4"/>Import</Button>
            </label>
          </div>
        </div>
      </header>

      <main className="max-w-6xl mx-auto px-4 py-6">
        <Tabs value={tab} onValueChange={setTab} className="w-full">
          <TabsList className="grid grid-cols-3 w-full">
            <TabsTrigger value="teacher" className="gap-2"><ClipboardList className="size-4"/>สำหรับครู</TabsTrigger>
            <TabsTrigger value="student" className="gap-2"><CheckCircle2 className="size-4"/>สำหรับนักเรียน</TabsTrigger>
            <TabsTrigger value="admin" className="gap-2"><BarChart3 className="size-4"/>แดชบอร์ดผู้บริหาร</TabsTrigger>
          </TabsList>

          {/* TEACHER */}
          <TabsContent value="teacher" className="mt-6 space-y-6">
            <Card className="shadow-sm border-rose-100">
              <CardHeader className="pb-3">
                <CardTitle>มอบหมายการบ้าน</CardTitle>
                <CardDescription>ระบุรายวิชา ระดับชั้น และกำหนดส่ง</CardDescription>
              </CardHeader>
              <CardContent className="space-y-4">
                <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                  <div>
                    <Label>รายวิชา</Label>
                    <Select value={subject} onValueChange={setSubject}>
                      <SelectTrigger><SelectValue placeholder="เลือกวิชา" /></SelectTrigger>
                      <SelectContent>
                        {SUBJECTS.map(s => (<SelectItem key={s} value={s}>{s}</SelectItem>))}
                      </SelectContent>
                    </Select>
                  </div>
                  <div>
                    <Label>ระดับชั้น</Label>
                    <Select value={classLevel} onValueChange={setClassLevel}>
                      <SelectTrigger><SelectValue placeholder="เลือกระดับชั้น" /></SelectTrigger>
                      <SelectContent>
                        {CLASS_LEVELS.map(c => (<SelectItem key={c} value={c}>{c}</SelectItem>))}
                      </SelectContent>
                    </Select>
                  </div>
                  <div>
                    <Label>หัวข้อการบ้าน</Label>
                    <Input value={title} onChange={e=>setTitle(e.target.value)} placeholder="เช่น แบบฝึกหัดหน้าที่ 45 ข้อ 1–10" />
                  </div>
                </div>

                <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                  <div>
                    <Label className="flex items-center gap-2"><CalendarDays className="size-4"/>วันที่มอบ</Label>
                    <Input type="date" value={assignedDate} onChange={e=>setAssignedDate(e.target.value)} />
                  </div>
                  <div>
                    <Label className="flex items-center gap-2"><CalendarDays className="size-4"/>กำหนดส่ง</Label>
                    <Input type="date" value={dueDate} onChange={e=>setDueDate(e.target.value)} />
                  </div>
                  <div>
                    <Label>แนบลิงก์</Label>
                    <Input value={link} onChange={e=>setLink(e.target.value)} placeholder="เช่น Google Doc / YouTube / Classroom"/>
                  </div>
                </div>

                <div>
                  <Label>คำอธิบาย</Label>
                  <Textarea value={description} onChange={e=>setDescription(e.target.value)} placeholder="รายละเอียด/เกณฑ์ให้คะแนน"/>
                </div>

                <div className="flex flex-wrap items-center gap-2">
                  <span className="text-sm text-slate-600">แม่แบบด่วน:</span>
                  {QUICK_TEMPLATES.map((q, idx) => (
                    <Button key={idx} type="button" size="sm" variant="secondary" className="rounded-full" onClick={()=>{ setTitle(q.title); if (q.description) setDescription(q.description); }}>
                      <PlusCircle className="size-4 mr-1"/>{q.title}
                    </Button>
                  ))}
                </div>

                <div className="flex items=center justify-between">
                  <p className="text-xs text-slate-500">ระบบจะบันทึกข้อมูลอัตโนมัติในอุปกรณ์นี้ (ต้นแบบ)</p>
                  <Button className="rounded-2xl" onClick={addHomework}>เพิ่มการบ้าน</Button>
                </div>
              </CardContent>
            </Card>

            <Card className="shadow-sm">
              <CardHeader className="pb-3">
                <div className="flex items-center justify-between gap-3">
                  <div>
                    <CardTitle>รายการการบ้านทั้งหมด</CardTitle>
                    <CardDescription>จัดเรียง/ค้นหาตามตัวกรองด้านขวา</CardDescription>
                  </div>
                  <FilterChips/>
                </div>
              </CardHeader>
              <CardContent>
                <Table>
                  <TableHeader>
                    <TableRow>
                      <TableHead>หัวข้อ</TableHead>
                      <TableHead>วิชา</TableHead>
                      <TableHead>ชั้น</TableHead>
                      <TableHead>มอบ</TableHead>
                      <TableHead>กำหนดส่ง</TableHead>
                      <TableHead className="text-right">ลบ</TableHead>
                    </TableRow>
                  </TableHeader>
                  <TableBody>
                    {teacherList.length === 0 && (
                      <TableRow>
                        <TableCell colSpan={6}><EmptyState/></TableCell>
                      </TableRow>
                    )}
                    {teacherList.map(i => (
                      <TableRow key={i.id} className="hover:bg-slate-50/60">
                        <TableCell>
                          <div className="font-medium">{i.title}</div>
                          <div className="text-xs text-slate-500 line-clamp-1">{i.description}</div>
                          {i.link && (
                            <a className="text-xs text-blue-600 hover:underline" href={i.link} target="_blank" rel="noreferrer">เปิดลิงก์</a>
                          )}
                        </TableCell>
                        <TableCell>{i.subject}</TableCell>
                        <TableCell>{i.classLevel}</TableCell>
                        <TableCell className="text-xs">{displayDate(i.assignedDate)}</TableCell>
                        <TableCell>
                          <div className="flex items-center gap-2">
                            <Badge variant={dueBadgeColor(daysUntil(i.dueDate)) as any}>
                              {displayDate(i.dueDate)}
                            </Badge>
                          </div>
                        </TableCell>
                        <TableCell className="text-right">
                          <Button variant="ghost" size="icon" onClick={()=>removeHomework(i.id)}>
                            <Trash2 className="size-4 text-rose-500"/>
                          </Button>
                        </TableCell>
                      </TableRow>
                    ))}
                  </TableBody>
                </Table>
              </CardContent>
            </Card>
          </TabsContent>

          {/* STUDENT */}
          <TabsContent value="student" className="mt-6">
            <Card className="shadow-sm">
              <CardHeader>
                <CardTitle className="flex items-center gap-2"><Users className="size-5"/>นักเรียนเช็คการบ้าน</CardTitle>
                <CardDescription>เลือกระดับชั้นและกรอกชื่อนักเรียนเพื่อบันทึกความคืบหน้าเฉพาะบุคคล</CardDescription>
              </CardHeader>
              <CardContent className="space-y-4">
                <div className="grid md:grid-cols-3 gap-4">
                  <div>
                    <Label>ระดับชั้น</Label>
                    <Select value={studentClass} onValueChange={setStudentClass}>
                      <SelectTrigger><SelectValue placeholder="เลือกระดับชั้น"/></SelectTrigger>
                      <SelectContent>
                        {CLASS_LEVELS.map(c => (<SelectItem key={c} value={c}>{c}</SelectItem>))}
                      </SelectContent>
                    </Select>
                  </div>
                  <div className="md:col-span-2">
                    <Label>ชื่อนักเรียน</Label>
                    <Input value={studentName} onChange={e=>setStudentName(e.target.value)} placeholder="พิมพ์ชื่อ–นามสกุล"/>
                    <p className="text-xs text-slate-500 mt-1">ระบบจะจดจำสถานะการบ้านตามชื่อที่กรอก</p>
                  </div>
                </div>
                <Separator/>
                <div className="flex items-center justify-between">
                  <div className="flex items-center gap-2">
                    <Checkbox id="showdone" checked={showCompleted} onCheckedChange={v=>setShowCompleted(Boolean(v))}/>
                    <Label htmlFor="showdone">แสดงรายการที่ทำแล้ว</Label>
                  </div>
                  <div className="text-sm text-slate-600">ทั้งหมด {filteredItems.length} งาน • แสดง {visibleItems.length}</div>
                </div>

                <div className="grid gap-3">
                  {visibleItems.length === 0 && (
                    <div className="text-center text-slate-500 py-10">ยังไม่มีการบ้านสำหรับชั้นนี้ หรือทำครบแล้ว 🎉</div>
                  )}

                  {visibleItems.map(i => {
                    const done = !!studentData[i.id];
                    const days = daysUntil(i.dueDate);
                    return (
                      <div key={i.id} className="p-4 rounded-2xl border bg-white hover:shadow-sm transition">
                        <div className="flex items-start justify-between gap-3">
                          <div>
                            <div className="flex items-center gap-2 flex-wrap">
                              <Badge variant="outline">{i.subject}</Badge>
                              <Badge variant={dueBadgeColor(days) as any}>
                                {days < 0 ? `เลยกำหนด ${Math.abs(days)} วัน` : days === 0 ? "ถึงกำหนดวันนี้" : `อีก ${days} วัน`}
                              </Badge>
                            </div>
                            <h3 className="mt-2 font-semibold text-base">{i.title}</h3>
                            {i.description && (
                              <p className="text-sm text-slate-600 mt-1">{i.description}</p>
                            )}
                            {i.link && (
                              <a href={i.link} target="_blank" rel="noreferrer" className="text-sm text-blue-600 hover:underline">เปิดลิงก์งาน</a>
                            )}
                            <p className="text-xs text-slate-500 mt-2">มอบหมาย {displayDate(i.assignedDate)} • กำหนดส่ง {displayDate(i.dueDate)}</p>
                          </div>
                          <div className="flex items-center gap-2">
                            <Checkbox id={`done-${i.id}`} checked={done} onCheckedChange={(v)=>toggleDone(studentName, i.id, Boolean(v))}/>
                            <Label htmlFor={`done-${i.id}`}>ทำแล้ว</Label>
                          </div>
                        </div>
                      </div>
                    );
                  })}
                </div>
              </CardContent>
            </Card>
          </TabsContent>

          {/* ADMIN */}
          <TabsContent value="admin" className="mt-6">
            <div className="grid md:grid-cols-2 gap-6">
              <Card className="shadow-sm">
                <CardHeader>
                  <CardTitle>จำนวนการบ้านตามรายวิชา</CardTitle>
                  <CardDescription>ดูภาพรวมปริมาณงานแต่ละวิชา</CardDescription>
                </CardHeader>
                <CardContent className="h-72">
                  {chartBySubject.length === 0 ? (
                    <div className="h-full flex items-center justify-center text-slate-500">ยังไม่มีข้อมูล</div>
                  ) : (
                    <ResponsiveContainer width="100%" height="100%">
                      <BarChart data={chartBySubject}>
                        <XAxis dataKey="name" fontSize={12} />
                        <YAxis allowDecimals={false} fontSize={12} />
                        <Tooltip />
                        <Bar dataKey="total" radius={[8,8,0,0]} />
                      </BarChart>
                    </ResponsiveContainer>
                  )}
                </CardContent>
              </Card>

              <Card className="shadow-sm">
                <CardHeader>
                  <CardTitle>สัดส่วนทำงานเสร็จโดยรวม (ประเมิน)</CardTitle>
                  <CardDescription>เฉลี่ยตามระดับชั้น</CardDescription>
                </CardHeader>
                <CardContent className="h-72">
                  {completionByClass.length === 0 ? (
                    <div className="h-full flex items-center justify-center text-slate-500">ยังไม่มีข้อมูล</div>
                  ) : (
                    <ResponsiveContainer width="100%" height="100%">
                      <BarChart data={completionByClass}>
                        <XAxis dataKey="name" fontSize={12} />
                        <YAxis domain={[0,100]} tickFormatter={(v)=>`${v}%`} fontSize={12} />
                        <Tooltip formatter={(v)=>`${v}%`} />
                        <Bar dataKey="percent" radius={[8,8,0,0]} />
                      </BarChart>
                    </ResponsiveContainer>
                  )}
                </CardContent>
              </Card>
            </div>

            <Card className="shadow-sm mt-6">
              <CardHeader>
                <CardTitle>แนวทางเชื่อมต่อระบบโรงเรียน (Roadmap)</CardTitle>
                <CardDescription>เมื่อย้ายจากต้นแบบสู่ระบบใช้งานจริง</CardDescription>
              </CardHeader>
              <CardContent className="text-sm leading-relaxed text-slate-700 space-y-2">
                <ul className="list-disc pl-5 space-y-1">
                  <li>Single Sign-On (SSO) ด้วยบัญชี Google Workspace @skv.ac.th</li>
                  <li>เชื่อมฐานข้อมูล (Firebase/Firestore) สำหรับเก็บงานและสถานะ</li>
                  <li>สิทธิ์การเข้าถึง: ครูเห็นเฉพาะชั้น/วิชาตนเอง, ผู้บริหารเห็นภาพรวม</li>
                  <li>นโยบาย PDPA: แสดง Privacy Notice, เก็บข้อมูลเท่าที่จำเป็น</li>
                  <li>สำรองข้อมูลอัตโนมัติรายวัน และ Audit Logs</li>
                </ul>
              </CardContent>
            </Card>
          </TabsContent>
        </Tabs>
      </main>

      <footer className="max-w-6xl mx-auto px-4 py-10 text-center text-xs text-slate-500">
        © {new Date().getFullYear()} Suankularb Wittayalai • ICT Smart School Initiative
      </footer>
    </div>
  );
}
