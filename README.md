# dialflow-crm
// app.js - DialFlow CRM Frontend
// TruIntel Reform Foundation - Admin + Agent Panels

let currentUser = null;
let allContacts = [];
let filteredContacts = [];
let currentCallIndex = 0;
let isAutoDialing = false;
let callTimer = null;
let callDuration = 0;
let currentContact = null;
let selectedStatus = '';
let recognition = null;

// DOM refs
const E = {
    ls: document.getElementById('ls'), ac: document.getElementById('ac'),
    lu: document.getElementById('lu'), lp: document.getElementById('lp'), lb: document.getElementById('lb'),
    urb: document.getElementById('urb'), und: document.getElementById('und'), lob: document.getElementById('lob'),
    an: document.getElementById('an'), ap: document.getElementById('ap'),
    adash: document.getElementById('adash'), acon: document.getElementById('acon'), aup: document.getElementById('aup'), aan: document.getElementById('aan'),
    si: document.getElementById('si'), vsb: document.getElementById('vsb'), vs: document.getElementById('vs'),
    sad: document.getElementById('sad'), cl: document.getElementById('cl'), cc: document.getElementById('cc'), es: document.getElementById('es'),
    mtc: document.getElementById('mtc'), mtt: document.getElementById('mtt'), ml: document.getElementById('ml'),
    // Active call
    acs: document.getElementById('acs'), ci: document.getElementById('ci'), cn: document.getElementById('cn'), cj: document.getElementById('cj'),
    cp: document.getElementById('cp'), cpl: document.getElementById('cpl'), ce: document.getElementById('ce'), clr: document.getElementById('clr'),
    ct: document.getElementById('ct'), csb: document.getElementById('csb'), ecb: document.getElementById('ecb'),
    // Disposition
    dm: document.getElementById('dm'), mcn: document.getElementById('mcn'), mcd: document.getElementById('mcd'), cn2: document.getElementById('cn2'),
    sdb: document.getElementById('sdb'), nct: document.getElementById('nct'), cd: document.getElementById('cd'),
    // Add contact
    acm: document.getElementById('acm'), cacb: document.getElementById('cacb'), ncn: document.getElementById('ncn'), nce: document.getElementById('nce'),
    ncp: document.getElementById('ncp'), ncj: document.getElementById('ncj'), sncb: document.getElementById('sncb'),
    // History
    hm: document.getElementById('hm'), hcn: document.getElementById('hcn'), hl: document.getElementById('hl'), chb: document.getElementById('chb'),
    // Admin
    atb: document.querySelectorAll('.atb'), actb: document.getElementById('actb'), acb: document.getElementById('acb'),
    atc: document.getElementById('atc'), atca: document.getElementById('atca'), aconv: document.getElementById('aconv'), att: document.getElementById('att'),
    aint: document.getElementById('aint'), abus: document.getElementById('abus'), ani: document.getElementById('ani'),
    cfi: document.getElementById('cfi'), ures: document.getElementById('ures'), urt: document.getElementById('urt'), uhl: document.getElementById('uhl'),
    // Toast
    t: document.getElementById('t'), tm: document.getElementById('tm')
};

document.addEventListener('DOMContentLoaded', () => {
    initEventListeners();
    setupVoiceSearch();
    let lastTouch = 0;
    document.addEventListener('touchend', e => { const n = Date.now(); if (n - lastTouch <= 300) e.preventDefault(); lastTouch = n; }, { passive: false });
});

// ===== AUTH =====
function initEventListeners() {
    E.lb.addEventListener('click', doLogin);
    E.lob.addEventListener('click', doLogout);
    E.si.addEventListener('input', e => filterContacts(e.target.value));
    E.vsb.addEventListener('click', toggleVoiceSearch);
    E.sad.addEventListener('click', () => isAutoDialing ? stopAutoDialing() : startAutoDialing());
    E.ecb.addEventListener('click', endCurrentCall);
    E.sdb.addEventListener('click', saveDisposition);
    E.chb.addEventListener('click', hideHistory);
    E.hm.addEventListener('click', e => { if (e.target === E.hm) hideHistory(); });
    E.dm.addEventListener('click', e => { if (e.target === E.dm) hideDisposition(); });
    E.cacb.addEventListener('click', hideAddContact);
    E.acm.addEventListener('click', e => { if (e.target === E.acm) hideAddContact(); });
    E.acb.addEventListener('click', showAddContact);
    E.sncb.addEventListener('click', saveNewContact);
    E.cfi.addEventListener('change', handleCSVUpload);

    document.querySelectorAll('.stb').forEach(btn => {
        btn.addEventListener('click', () => {
            selectedStatus = btn.dataset.status;
            document.querySelectorAll('.stb').forEach(b => b.classList.remove('ring-2', 'ring-offset-2'));
            btn.classList.add('ring-2', 'ring-offset-2');
            const map = { Interested: 'ring-emerald-500', Busy: 'ring-amber-500', 'Not Interested': 'ring-red-500' };
            btn.classList.add(map[selectedStatus]);
        });
    });

    E.atb.forEach(tab => {
        tab.addEventListener('click', () => switchAdminTab(tab.dataset.tab));
    });

    document.addEventListener('keydown', e => { if (e.key === 'Escape') { hideDisposition(); hideHistory(); hideAddContact(); } });
}

async function doLogin() {
    const username = E.lu.value.trim();
    const password = E.lp.value.trim();
    if (!username || !password) { toast('Enter username and password', true); return; }
    try {
        const res = await fetch('/api/login', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ username, password }) });
        const data = await res.json();
        if (data.success) {
            currentUser = data.user;
            E.ls.classList.add('hidden');
            E.ac.classList.remove('hidden');
            E.urb.textContent = currentUser.role === 'admin' ? 'Manager' : 'Agent';
            E.und.textContent = currentUser.name;
            if (currentUser.role === 'admin') {
                E.an.classList.remove('hidden');
                E.ap.classList.add('hidden');
                switchAdminTab('dashboard');
                loadAdminData();
            } else {
                E.an.classList.add('hidden');
                E.ap.classList.remove('hidden');
                loadAgentData();
            }
            toast(`Welcome, ${currentUser.name}!`);
        } else {
            toast('Invalid credentials', true);
        }
    } catch (e) { toast('Login error', true); }
}

function doLogout() {
    currentUser = null;
    E.ac.classList.add('hidden');
    E.ls.classList.remove('hidden');
    E.lu.value = ''; E.lp.value = '';
    stopAutoDialing();
}

// ===== ADMIN =====
function switchAdminTab(tab) {
    E.atb.forEach(t => {
        if (t.dataset.tab === tab) { t.classList.add('text-blue-600', 'border-blue-600'); t.classList.remove('text-slate-500', 'border-transparent'); }
        else { t.classList.remove('text-blue-600', 'border-blue-600'); t.classList.add('text-slate-500', 'border-transparent'); }
    });
    ['dashboard', 'contacts', 'upload', 'analytics'].forEach(t => {
        document.getElementById('a' + (t === 'dashboard' ? 'dash' : t === 'contacts' ? 'con' : t === 'upload' ? 'up' : 'an')).classList.add('hidden');
    });
    document.getElementById('a' + (tab === 'dashboard' ? 'dash' : tab === 'contacts' ? 'con' : tab === 'upload' ? 'up' : 'an')).classList.remove('hidden');
    if (tab === 'dashboard') loadAdminAnalytics();
    if (tab === 'contacts') loadAdminContacts();
    if (tab === 'upload') loadUploadHistory();
}

async function loadAdminData() { loadAdminAnalytics(); loadAdminContacts(); loadUploadHistory(); }

async function loadAdminAnalytics() {
    try {
        const res = await fetch('/api/admin/analytics');
        const data = await res.json();
        if (data.success) {
            const a = data.analytics;
            E.atc.textContent = a.totalContacts;
            E.atca.textContent = a.totalCalls;
            E.aconv.textContent = a.conversionRate + '%';
            E.att.textContent = Math.round(a.totalTalkTime / 60) + 'm';
            E.aint.textContent = a.interested;
            E.abus.textContent = a.busy;
            E.ani.textContent = a.notInterested;
        }
    } catch (e) { console.error(e); }
}

async function loadAdminContacts() {
    try {
        const res = await fetch('/api/admin/contacts');
        const data = await res.json();
        if (data.success) {
            E.actb.innerHTML = data.contacts.map(c => `
                <tr class="hover:bg-slate-50">
                    <td class="px-3 sm:px-4 py-3">
                        <div class="flex items-center space-x-2">
                            <div class="w-8 h-8 bg-blue-100 rounded-full flex items-center justify-center text-blue-700 font-bold text-xs">${gi(c.name)}</div>
                            <span class="font-medium text-slate-800">${c.name}</span>
                        </div>
                    </td>
                    <td class="px-3 sm:px-4 py-3 text-slate-600">${c.phone}</td>
                    <td class="px-3 sm:px-4 py-3 text-slate-600 hidden md:table-cell">${c.email}</td>
                    <td class="px-3 sm:px-4 py-3 text-slate-600 hidden lg:table-cell">${c.jobTitle || '-'}</td>
                    <td class="px-3 sm:px-4 py-3"><span class="px-2 py-0.5 text-[10px] rounded-full ${gs(c.status)}">${c.status}</span></td>
                    <td class="px-3 sm:px-4 py-3 text-slate-600">${c.totalCalls}</td>
                    <td class="px-3 sm:px-4 py-3 text-right">
                        <button onclick="deleteContact(${c.id})" class="text-red-500 hover:text-red-700 text-xs"><i class="fas fa-trash"></i></button>
                    </td>
                </tr>
            `).join('');
        }
    } catch (e) { console.error(e); }
}

async function saveNewContact() {
    const name = E.ncn.value.trim(), email = E.nce.value.trim(), phone = E.ncp.value.trim(), jobTitle = E.ncj.value.trim();
    if (!name || !email || !phone) { toast('Fill required fields', true); return; }
    try {
        const res = await fetch('/api/admin/contacts', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ name, email, phone, jobTitle }) });
        const data = await res.json();
        if (data.success) { toast('Contact added'); hideAddContact(); loadAdminContacts(); E.ncn.value = ''; E.nce.value = ''; E.ncp.value = ''; E.ncj.value = ''; }
        else toast('Failed to add', true);
    } catch (e) { toast('Error', true); }
}

async function deleteContact(id) {
    if (!confirm('Delete this contact?')) return;
    try {
        const res = await fetch(`/api/admin/contacts/${id}`, { method: 'DELETE' });
        if (res.ok) { toast('Contact deleted'); loadAdminContacts(); }
    } catch (e) { toast('Error', true); }
}

async function handleCSVUpload(e) {
    const file = e.target.files[0];
    if (!file) return;
    const formData = new FormData();
    formData.append('csvFile', file);
    try {
        const res = await fetch('/api/admin/upload-csv', { method: 'POST', body: formData });
        const data = await res.json();
        if (data.success) {
            E.ures.classList.remove('hidden');
            E.urt.textContent = `${data.imported} contacts imported successfully`;
            loadUploadHistory(); loadAdminContacts();
        } else { toast('Upload failed', true); }
    } catch (e) { toast('Upload error', true); }
    e.target.value = '';
}

async function loadUploadHistory() {
    try {
        const res = await fetch('/api/admin/uploads');
        const data = await res.json();
        if (data.success) {
            E.uhl.innerHTML = (data.uploads || []).length === 0 ? '<p class="text-sm text-slate-500 text-center py-4">No uploads yet</p>' :
                data.uploads.slice().reverse().map(u => `
                    <div class="bg-white rounded-lg shadow-sm border border-slate-200 p-3 flex items-center justify-between">
                        <div class="flex items-center space-x-2">
                            <i class="fas fa-file-csv text-blue-600"></i>
                            <div>
                                <p class="text-xs font-medium text-slate-800">${u.filename}</p>
                                <p class="text-[10px] text-slate-500">${new Date(u.date).toLocaleDateString()}</p>
                            </div>
                        </div>
                        <span class="text-xs font-semibold text-emerald-600 bg-emerald-50 px-2 py-1 rounded">+${u.count}</span>
                    </div>
                `).join('');
        }
    } catch (e) { console.error(e); }
}

function showAddContact() { E.acm.classList.remove('hidden'); }
function hideAddContact() { E.acm.classList.add('hidden'); }

// ===== AGENT =====
async function loadAgentData() {
    await loadContacts();
    await loadDashboardMetrics();
}

async function loadContacts() {
    try {
        const res = await fetch('/api/contacts');
        const data = await res.json();
        if (data.success) {
            allContacts = data.contacts;
            filteredContacts = [...allContacts];
            renderContacts();
            updateCount();
        }
    } catch (e) { toast('Error loading contacts', true); }
}

async function loadDashboardMetrics() {
    try {
        const res = await fetch('/api/dashboard');
        const data = await res.json();
        if (data.success) updateMetrics(data.metrics);
    } catch (e) { console.error(e); }
}

async function logCall(contactId, duration, status, notes) {
    try {
        const res = await fetch('/api/log-call', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ contactId, duration, status, notes, timestamp: new Date().toISOString() }) });
        const data = await res.json();
        if (data.success) { toast('Call logged'); loadDashboardMetrics(); loadContacts(); }
    } catch (e) { toast('Error saving', true); }
}

async function loadCallHistory(contactId, contactName) {
    try {
        const res = await fetch(`/api/call-history/${contactId}`);
        const data = await res.json();
        if (data.success) renderHistory(data.history, contactName);
    } catch (e) { toast('Error loading history', true); }
}

function renderContacts() {
    if (filteredContacts.length === 0) { E.cl.innerHTML = ''; E.es.classList.remove('hidden'); return; }
    E.es.classList.add('hidden');
    E.cl.innerHTML = filteredContacts.map(c => {
        const hasReport = c.lastCallReport && c.lastCallReport !== 'No previous calls';
        return `<div class="bg-white rounded-xl shadow-sm border border-slate-200 p-3 sm:p-4 ch" data-id="${c.id}">
            <div class="flex items-start space-x-3">
                <div class="w-10 h-10 sm:w-11 sm:h-11 bg-gradient-to-br from-blue-600 to-blue-800 rounded-full flex items-center justify-center text-white font-bold text-sm sm:text-base shrink-0 shadow-sm">${gi(c.name)}</div>
                <div class="flex-1 min-w-0">
                    <div class="flex items-start justify-between gap-2">
                        <div class="min-w-0">
                            <h3 class="font-semibold text-slate-800 text-sm sm:text-base truncate">${c.name}</h3>
                            <p class="text-[10px] sm:text-xs text-slate-500 truncate">${c.jobTitle || 'No title'}</p>
                            <div class="flex flex-wrap items-center gap-x-3 gap-y-0.5 mt-1 text-[10px] sm:text-xs text-slate-500">
                                <span class="flex items-center truncate max-w-[140px]"><i class="fas fa-envelope mr-1 text-slate-400 text-[8px]"></i>${c.email}</span>
                                <a href="tel:${c.phone.replace(/[^0-9+]/g, '')}" class="flex items-center text-blue-600 font-medium"><i class="fas fa-phone mr-1 text-slate-400 text-[8px]"></i>${c.phone}</a>
                            </div>
                            ${hasReport ? `<div class="mt-1.5 flex items-start space-x-1.5 bg-amber-50 rounded-lg px-2 py-1"><i class="fas fa-file-alt text-amber-500 text-[8px] mt-0.5"></i><p class="text-[10px] text-amber-700 truncate">${c.lastCallReport}</p></div>` : ''}
                        </div>
                        <div class="flex flex-col space-y-1.5 shrink-0">
                            ${c.status !== 'New' ? `<span class="px-1.5 py-0.5 text-[9px] rounded-full ${gs(c.status)} text-center">${c.status}</span>` : ''}
                            <button onclick="viewHistory(${c.id}, '${c.name}')" class="w-8 h-8 bg-slate-100 hover:bg-slate-200 active:bg-slate-300 rounded-lg flex items-center justify-center transition"><i class="fas fa-history text-slate-600 text-xs"></i></button>
                            <button onclick="manualCall(${c.id})" class="w-8 h-8 bg-gradient-to-r from-blue-600 to-blue-700 hover:from-blue-700 hover:to-blue-800 active:from-blue-800 active:to-blue-900 rounded-lg flex items-center justify-center transition shadow-sm"><i class="fas fa-phone text-white text-xs"></i></button>
                        </div>
                    </div>
                </div>
            </div>
        </div>`;
    }).join('');
}

function renderHistory(history, contactName) {
    E.hcn.textContent = contactName;
    if (history.length === 0) {
        E.hl.innerHTML = `<div class="text-center py-8"><div class="w-12 h-12 bg-slate-100 rounded-full flex items-center justify-center mx-auto mb-3"><i class="fas fa-inbox text-slate-400 text-lg"></i></div><p class="text-sm text-slate-500">No history.</p></div>`;
        return;
    }
    E.hl.innerHTML = history.map(h => `
        <div class="border-l-4 ${gb(h.status)} pl-3 sm:pl-4 py-2.5 sm:py-3 mb-2.5 sm:mb-3 bg-slate-50 rounded-r-lg">
            <div class="flex items-center justify-between mb-1"><span class="font-semibold text-slate-800 text-xs sm:text-sm">${h.status}</span><span class="text-[10px] text-slate-500">${fd(h.timestamp)}</span></div>
            <p class="text-xs text-slate-600 mb-0.5"><i class="fas fa-clock mr-1 text-slate-400 text-[8px]"></i>${fmt(h.duration)}</p>
            ${h.notes ? `<p class="text-[11px] text-slate-500 italic mt-1">"${h.notes}"</p>` : ''}
        </div>
    `).join('');
}

function updateMetrics(m) { an(E.mtc, m.totalCalls); an(E.mtt, Math.round(m.totalTalkTime / 60)); an(E.ml, m.successfulLeads); }
function updateCount() { E.cc.textContent = filteredContacts.length; }

// ===== CALL LOGIC =====
function showActiveCall(contact, seq, total) {
    E.ci.textContent = gi(contact.name);
    E.cn.textContent = contact.name;
    E.cj.textContent = contact.jobTitle || 'No title';
    E.cp.textContent = contact.phone;
    E.cpl.href = `tel:${contact.phone.replace(/[^0-9+]/g, '')}`;
    E.ce.textContent = contact.email;
    E.clr.textContent = contact.lastCallReport || 'No previous calls.';
    E.csb.textContent = `${seq} / ${total}`;
    callDuration = 0; E.ct.textContent = '00:00';
    E.acs.classList.remove('hidden');
    document.body.style.overflow = 'hidden';
    callTimer = setInterval(() => { callDuration++; E.ct.textContent = fmt(callDuration); }, 1000);
    if (navigator.vibrate) navigator.vibrate([200, 100, 200]);
}

function hideActiveCall() { if (callTimer) { clearInterval(callTimer); callTimer = null; } E.acs.classList.add('hidden'); document.body.style.overflow = ''; }

function startAutoDialing() {
    if (isAutoDialing) return;
    isAutoDialing = true; currentCallIndex = 0;
    E.sad.innerHTML = '<i class="fas fa-stop"></i><span>Stop</span>';
    E.sad.classList.remove('ta'); E.sad.classList.add('bg-red-600');
    toast('Auto-dialing started!'); processNext();
}

function stopAutoDialing() {
    isAutoDialing = false;
    E.sad.innerHTML = '<i class="fas fa-phone-volume"></i><span>Auto-Dial</span>';
    E.sad.classList.add('ta'); E.sad.classList.remove('bg-red-600');
    hideActiveCall(); toast('Stopped');
}

function processNext() {
    if (!isAutoDialing || currentCallIndex >= filteredContacts.length) {
        stopAutoDialing();
        if (currentCallIndex >= filtered
