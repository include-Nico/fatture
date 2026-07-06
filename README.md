/* =======================================================
   GESTIONALE FATTURE - versione locale (senza cloud/Google Sheets)
   Tutti i dati (clienti, fatture, magazzino, impostazioni azienda)
   vengono salvati esclusivamente nel localStorage del browser.
   Usa Impostazioni > Backup per esportare/importare i dati.
   ======================================================= */

const STORAGE_KEY_DATA = 'invoiceAppDataBackup';
const STORAGE_KEY_THEME = 'invoiceAppTheme';

const DEFAULT_SETTINGS = {
    ragioneSociale: '',
    piva: '',
    cf: '',
    indirizzo: '',
    email: '',
    telefono: '',
    notaFiscale: '',
    notaLegale: '',
    notaDispositivo: '',
    logo: '',   // dataURL base64
    firma: ''   // dataURL base64
};

let appData = JSON.parse(localStorage.getItem(STORAGE_KEY_DATA)) || {
    clients: [],
    invoices: [],
    inventory: [],
    settings: { ...DEFAULT_SETTINGS }
};
if (!appData.settings) appData.settings = { ...DEFAULT_SETTINGS };
if (!appData.inventory) appData.inventory = [];

let invoiceItems = [];
let revenueChartInstance = null;
let currentFilteredInvoices = [];

const PLACEHOLDER_LOGO = 'assets/logo-placeholder.svg';

// --- INIZIALIZZAZIONE ---
document.addEventListener('DOMContentLoaded', () => {
    if (localStorage.getItem(STORAGE_KEY_THEME) === 'dark') {
        document.documentElement.setAttribute('data-theme', 'dark');
    }

    document.getElementById('input-data').valueAsDate = new Date();

    loadSettingsForm();
    applyBranding();
    refreshAll();
    initConformita();

    setSaveStatus('success');
});

function showToast(message, type = 'success') {
    const container = document.getElementById('toast-container');
    const toast = document.createElement('div');
    toast.className = `toast ${type}`;
    const icon = type === 'success' ? '<i class="fas fa-check-circle"></i>' : '<i class="fas fa-exclamation-circle"></i>';
    toast.innerHTML = `${icon} <span>${message}</span>`;
    container.appendChild(toast);
    setTimeout(() => {
        toast.style.animation = 'fadeOut 0.3s ease forwards';
        setTimeout(() => toast.remove(), 300);
    }, 3000);
}

// Indicatore di stato salvataggio locale (sostituisce il vecchio stato "cloud")
function setSaveStatus(status) {
    const icons = [document.getElementById('cloud-mobile'), document.getElementById('cloud-desktop')];
    icons.forEach(span => {
        if (!span) return;
        if (status === 'saving') span.innerHTML = '<i class="fas fa-sync-alt fa-spin" style="color: #f39c12;"></i><span class="desktop-only" style="margin-left:5px;"> Salvataggio...</span>';
        if (status === 'success') span.innerHTML = '<i class="fas fa-check-circle" style="color: #2ecc71;"></i><span class="desktop-only" style="margin-left:5px;"> Salvato in locale</span>';
        if (status === 'error') span.innerHTML = '<i class="fas fa-exclamation-circle" style="color: #e74c3c;"></i><span class="desktop-only" style="margin-left:5px;"> Errore salvataggio</span>';
    });
}

function toggleTheme() {
    const currentTheme = document.documentElement.getAttribute('data-theme');
    const newTheme = currentTheme === 'dark' ? 'light' : 'dark';
    document.documentElement.setAttribute('data-theme', newTheme);
    localStorage.setItem(STORAGE_KEY_THEME, newTheme);
    updateChart();
}

function calculateNextInvoiceNumber(dateString) {
    const year = new Date(dateString || new Date()).getFullYear();
    const yearInvoices = appData.invoices.filter(i => new Date(i.data).getFullYear() === year);
    const maxNum = yearInvoices.reduce((max, inv) => Math.max(max, inv.numero), 0);
    return maxNum + 1;
}

function handleDateChange() {
    document.getElementById('input-numero').value = calculateNextInvoiceNumber(document.getElementById('input-data').value);
    updatePreview();
}

function refreshAll() {
    document.getElementById('input-numero').value = calculateNextInvoiceNumber(document.getElementById('input-data').value);
    refreshClientsTable();
    refreshClientsDropdown();
    populateYearsFilters();
    refreshInvoicesTable();
    refreshInventoryTable();
    initInvoiceTab();
}

// Salvataggio 100% locale: nessuna chiamata di rete.
function saveData() {
    setSaveStatus('saving');
    try {
        localStorage.setItem(STORAGE_KEY_DATA, JSON.stringify(appData));
        setSaveStatus('success');
    } catch (err) {
        console.error('Errore salvataggio locale:', err);
        setSaveStatus('error');
        showToast('Errore nel salvataggio locale (spazio browser pieno?)', 'error');
    }
    refreshAll();
}

function populateYearsFilters() {
    const years = new Set(appData.invoices.map(i => new Date(i.data).getFullYear()));
    const currentYear = new Date().getFullYear();
    years.add(currentYear);

    const sortedYears = Array.from(years).sort((a, b) => b - a);

    const filterYearSelect = document.getElementById('filter-inv-year');
    const chartYearSelect = document.getElementById('chart-year');

    const currFilterVal = filterYearSelect.value;
    const currChartVal = chartYearSelect.value;

    filterYearSelect.innerHTML = '<option value="">Tutti gli anni</option>';
    chartYearSelect.innerHTML = '';

    sortedYears.forEach(y => {
        filterYearSelect.innerHTML += `<option value="${y}">${y}</option>`;
        chartYearSelect.innerHTML += `<option value="${y}">${y}</option>`;
    });

    if (currFilterVal) filterYearSelect.value = currFilterVal;
    if (currChartVal) { chartYearSelect.value = currChartVal; } else { chartYearSelect.value = currentYear; }
}

function toggleSidebar() {
    document.getElementById('sidebar').classList.toggle('open');
    document.getElementById('sidebar-overlay').classList.toggle('active');
}

function switchTab(tabId, btn) {
    document.querySelectorAll('.tab-section').forEach(el => el.classList.add('hidden'));
    document.getElementById(tabId).classList.remove('hidden');
    document.querySelectorAll('.nav-btn').forEach(el => el.classList.remove('active'));
    if (btn) btn.classList.add('active');
    if (window.innerWidth <= 850) { toggleSidebar(); }

    if (tabId === 'tab-fattura') initInvoiceTab();
    if (tabId === 'tab-conformita') initConformita();
    if (tabId === 'tab-impostazioni') loadSettingsForm();
}

function toggleMobilePreview() {
    document.getElementById('mobile-preview-container').classList.toggle('show-mobile');
}

function toggleConformityPreview() {
    const previewPanel = document.getElementById('conformity-mobile-preview-container');
    if (previewPanel) {
        previewPanel.classList.toggle('show-mobile');
    }
}

function toggleInvoiceFilters() {
    document.getElementById('invoice-filters').classList.toggle('hidden');
}

function openWhatsApp(phone) {
    if (!phone) { showToast("Numero mancante", "error"); return; }
    let cleanPhone = phone.replace(/\D/g, '');
    if (!cleanPhone.startsWith('39') && cleanPhone.length === 10) cleanPhone = '39' + cleanPhone;
    let msg = encodeURIComponent("...");
    window.open(`https://wa.me/${cleanPhone}?text=${msg}`, '_blank');
}

/* ======================================================= */
/* IMPOSTAZIONI AZIENDA (sostituisce Google Sheets)         */
/* ======================================================= */

function loadSettingsForm() {
    const s = appData.settings || DEFAULT_SETTINGS;
    const setVal = (id, val) => { const el = document.getElementById(id); if (el) el.value = val || ''; };

    setVal('set-ragione-sociale', s.ragioneSociale);
    setVal('set-piva', s.piva);
    setVal('set-cf', s.cf);
    setVal('set-indirizzo', s.indirizzo);
    setVal('set-email', s.email);
    setVal('set-telefono', s.telefono);
    setVal('set-nota-fiscale', s.notaFiscale);
    setVal('set-nota-legale', s.notaLegale);
    setVal('set-nota-dispositivo', s.notaDispositivo);

    updateImagePreviewBox('logo-preview-box', s.logo, 'fa-image');
    updateImagePreviewBox('firma-preview-box', s.firma, 'fa-signature');
}

function updateImagePreviewBox(boxId, dataUrl, iconClass) {
    const box = document.getElementById(boxId);
    if (!box) return;
    if (dataUrl) {
        box.innerHTML = `<img src="${dataUrl}" alt="preview">`;
    } else {
        box.innerHTML = `<i class="fas ${iconClass}"></i>`;
    }
}

function handleImageUpload(event, kind) {
    const file = event.target.files && event.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = function (e) {
        const dataUrl = e.target.result;
        if (kind === 'logo') {
            updateImagePreviewBox('logo-preview-box', dataUrl, 'fa-image');
        } else {
            updateImagePreviewBox('firma-preview-box', dataUrl, 'fa-signature');
        }
        // Salva subito l'immagine nelle impostazioni correnti (verrà persistita al submit del form,
        // ma la teniamo anche pronta in caso l'utente non prema "Salva")
        if (!appData.settings) appData.settings = { ...DEFAULT_SETTINGS };
        appData.settings[kind] = dataUrl;
    };
    reader.readAsDataURL(file);
}

function removeBrandImage(kind) {
    if (!appData.settings) appData.settings = { ...DEFAULT_SETTINGS };
    appData.settings[kind] = '';
    if (kind === 'logo') {
        updateImagePreviewBox('logo-preview-box', '', 'fa-image');
    } else {
        updateImagePreviewBox('firma-preview-box', '', 'fa-signature');
    }
    saveData();
    applyBranding();
    showToast(kind === 'logo' ? 'Logo rimosso.' : 'Firma rimossa.');
}

function saveSettings(e) {
    e.preventDefault();
    const getVal = (id) => { const el = document.getElementById(id); return el ? el.value.trim() : ''; };

    if (!appData.settings) appData.settings = { ...DEFAULT_SETTINGS };

    appData.settings.ragioneSociale = getVal('set-ragione-sociale');
    appData.settings.piva = getVal('set-piva');
    appData.settings.cf = getVal('set-cf').toUpperCase();
    appData.settings.indirizzo = getVal('set-indirizzo');
    appData.settings.email = getVal('set-email');
    appData.settings.telefono = getVal('set-telefono');
    appData.settings.notaFiscale = getVal('set-nota-fiscale');
    appData.settings.notaLegale = getVal('set-nota-legale');
    appData.settings.notaDispositivo = getVal('set-nota-dispositivo');
    // logo/firma sono già stati aggiornati in appData.settings da handleImageUpload/removeBrandImage

    saveData();
    applyBranding();
    showToast('Impostazioni salvate!');
}

// Applica i dati aziendali (nome, logo, firma, note) a tutte le viste dell'app
function applyBranding() {
    const s = appData.settings || DEFAULT_SETTINGS;
    const nome = s.ragioneSociale || 'La Tua Azienda';
    const logoSrc = s.logo || PLACEHOLDER_LOGO;

    // Sidebar / Header mobile e desktop
    const setText = (id, val) => { const el = document.getElementById(id); if (el) el.innerText = val; };
    const setImg = (id, src) => { const el = document.getElementById(id); if (el) el.src = src; };

    setText('app-name-mobile', nome.length > 14 ? nome.substring(0, 14) + '…' : nome);
    setText('app-name-desktop', nome);
    setImg('logo-img-mobile', logoSrc);
    setImg('logo-img-desktop', logoSrc);

    // Intestazione fattura
    setImg('invoice-logo-img', logoSrc);
    setText('prev-sup-nome', nome);
    const pivaCfParts = [];
    if (s.piva) pivaCfParts.push('P.IVA: ' + s.piva);
    if (s.cf) pivaCfParts.push('C.F.: ' + s.cf);
    setText('prev-sup-piva-cf', pivaCfParts.length ? pivaCfParts.join(' | ') : 'P.IVA / C.F. non impostati');
    setText('prev-sup-indirizzo', s.indirizzo || 'Indirizzo non impostato');
    setText('prev-sup-email', s.email ? ('Email: ' + s.email + (s.telefono ? ' | Tel: ' + s.telefono : '')) : 'Configura i dati aziendali nella scheda "Impostazioni"');

    const notaEl = document.getElementById('prev-sup-nota');
    if (notaEl) {
        notaEl.innerText = s.notaFiscale || (s.ragioneSociale ? '' : 'Configura i dati aziendali nella scheda "Impostazioni"');
        notaEl.parentElement.style.display = notaEl.innerText ? 'block' : 'none';
    }

    const notaLegaleEl = document.getElementById('prev-nota-legale');
    if (notaLegaleEl) {
        notaLegaleEl.style.display = s.notaLegale ? 'block' : 'none';
        notaLegaleEl.innerHTML = s.notaLegale ? `<em>${s.notaLegale}</em>` : '';
    }
    const notaDispEl = document.getElementById('prev-nota-dispositivo');
    if (notaDispEl) {
        notaDispEl.style.display = s.notaDispositivo ? 'block' : 'none';
        notaDispEl.innerHTML = s.notaDispositivo ? `<strong>${s.notaDispositivo}</strong>` : '';
    }

    // Intestazione dichiarazione di conformità
    setImg('conf-logo-img', logoSrc);
    setText('prev-conf-sup-nome', nome);
    setText('prev-conf-sup-indirizzo', s.indirizzo || 'Indirizzo non impostato');
    setText('prev-conf-sup-piva', s.piva ? ('P.IVA: ' + s.piva) : 'P.IVA: -');
    setText('prev-conf-sup-contatti', `Email: ${s.email || '-'} | Tel. ${s.telefono || '-'}`);

    // Firma sulla dichiarazione di conformità
    const firmaImg = document.getElementById('firma-img');
    if (firmaImg) {
        if (s.firma) {
            firmaImg.src = s.firma;
            firmaImg.style.display = 'block';
        } else {
            firmaImg.removeAttribute('src');
            firmaImg.style.display = 'none';
        }
    }
}

/* ======================================================= */
/* BACKUP / RIPRISTINO DATI (sostituisce Google Sheets)     */
/* ======================================================= */

function exportBackup() {
    const dataStr = JSON.stringify(appData, null, 2);
    const blob = new Blob([dataStr], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    const stamp = new Date().toISOString().slice(0, 19).replace(/[:T]/g, '-');
    link.href = url;
    link.download = `backup-fatture-${stamp}.json`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
    showToast('Backup esportato!');
}

function importBackup(event) {
    const file = event.target.files && event.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = function (e) {
        try {
            const imported = JSON.parse(e.target.result);
            if (!imported || typeof imported !== 'object') throw new Error('Formato non valido');

            appData = {
                clients: Array.isArray(imported.clients) ? imported.clients : [],
                invoices: Array.isArray(imported.invoices) ? imported.invoices : [],
                inventory: Array.isArray(imported.inventory) ? imported.inventory : [],
                settings: { ...DEFAULT_SETTINGS, ...(imported.settings || {}) }
            };

            saveData();
            loadSettingsForm();
            applyBranding();
            initConformita();
            showToast('Backup importato con successo!');
        } catch (err) {
            console.error('Errore importazione backup:', err);
            showToast('File di backup non valido.', 'error');
        } finally {
            event.target.value = '';
        }
    };
    reader.readAsText(file);
}

function resetAllData() {
    if (confirm('Sei sicuro di voler cancellare TUTTI i dati (clienti, fatture, magazzino, impostazioni)? Questa azione è irreversibile. Ti consigliamo di esportare prima un backup.')) {
        appData = { clients: [], invoices: [], inventory: [], settings: { ...DEFAULT_SETTINGS } };
        saveData();
        loadSettingsForm();
        applyBranding();
        initConformita();
        showToast('Tutti i dati sono stati cancellati.');
    }
}

/* ======================================================= */
/* CLIENTI & RICETTE                                        */
/* ======================================================= */
function openClientModal(clientId = null) {
    document.getElementById('client-form').reset();
    document.getElementById('client-id').value = '';
    document.getElementById('modal-client-title').innerText = 'Aggiungi Cliente & Ricetta';

    if (clientId) {
        const client = appData.clients.find(c => c.id === clientId);
        if (client) {
            document.getElementById('modal-client-title').innerText = 'Modifica Cliente & Ricetta';
            document.getElementById('client-id').value = client.id;
            document.getElementById('c-nome').value = client.nome || '';
            document.getElementById('c-tipo').value = client.tipo || 'fisica';
            document.getElementById('c-cf').value = client.cf || '';
            document.getElementById('c-piva').value = client.piva || '';
            document.getElementById('c-sdi').value = client.sdi || '';
            document.getElementById('c-via').value = client.via || '';
            document.getElementById('c-comune').value = client.comune || '';
            document.getElementById('c-cap').value = client.cap || '';
            document.getElementById('c-prov').value = client.prov || '';
            document.getElementById('c-telefono').value = client.telefono || '';
            document.getElementById('c-note').value = client.note || '';

            if (client.ricetta) {
                const r = client.ricetta;
                ['dx', 'sx'].forEach(lato => {
                    ['l', 'i', 'v'].forEach(dist => {
                        document.getElementById(`${lato}-sf-${dist}`).value = r[`${lato}_sf_${dist}`] || '';
                        document.getElementById(`${lato}-cil-${dist}`).value = r[`${lato}_cil_${dist}`] || '';
                        document.getElementById(`${lato}-ax-${dist}`).value = r[`${lato}_ax_${dist}`] || '';
                    });
                });
                document.getElementById('dist-lontano').value = r.dist_lontano || '';
                document.getElementById('dist-vicino').value = r.dist_vicino || '';
            }
        }
    }
    document.getElementById('client-modal').classList.remove('hidden');
}

function closeClientModal() { document.getElementById('client-modal').classList.add('hidden'); }

function saveClient(e) {
    e.preventDefault();
    const idStr = document.getElementById('client-id').value;

    const ricetta = {
        dist_lontano: document.getElementById('dist-lontano').value,
        dist_vicino: document.getElementById('dist-vicino').value
    };
    ['dx', 'sx'].forEach(lato => {
        ['l', 'i', 'v'].forEach(dist => {
            ricetta[`${lato}_sf_${dist}`] = document.getElementById(`${lato}-sf-${dist}`).value;
            ricetta[`${lato}_cil_${dist}`] = document.getElementById(`${lato}-cil-${dist}`).value;
            ricetta[`${lato}_ax_${dist}`] = document.getElementById(`${lato}-ax-${dist}`).value;
        });
    });

    const clientData = {
        id: idStr ? idStr : Date.now().toString(),
        nome: document.getElementById('c-nome').value,
        tipo: document.getElementById('c-tipo').value,
        cf: document.getElementById('c-cf').value.toUpperCase(),
        piva: document.getElementById('c-piva').value,
        sdi: document.getElementById('c-sdi').value,
        via: document.getElementById('c-via').value,
        comune: document.getElementById('c-comune').value,
        cap: document.getElementById('c-cap').value,
        prov: document.getElementById('c-prov').value.toUpperCase(),
        telefono: document.getElementById('c-telefono').value,
        note: document.getElementById('c-note').value,
        ricetta: ricetta
    };

    if (idStr) {
        const idx = appData.clients.findIndex(c => c.id === idStr);
        if (idx !== -1) appData.clients[idx] = clientData;
    } else {
        appData.clients.push(clientData);
    }

    saveData();
    closeClientModal();
    initConformita();
    showToast("Cliente salvato con successo!");
}

function viewRicetta(clientId) {
    const client = appData.clients.find(c => c.id === clientId);
    if (!client) return;
    document.getElementById('view-ricetta-nome').innerText = client.nome;
    const r = client.ricetta || {};
    ['dx', 'sx'].forEach(lato => {
        ['l', 'i', 'v'].forEach(dist => {
            document.getElementById(`v-${lato}-sf-${dist}`).innerText = r[`${lato}_sf_${dist}`] || '-';
            document.getElementById(`v-${lato}-cil-${dist}`).innerText = r[`${lato}_cil_${dist}`] || '-';
            document.getElementById(`v-${lato}-ax-${dist}`).innerText = r[`${lato}_ax_${dist}`] || '-';
        });
    });
    document.getElementById('v-dist-lontano').innerText = r.dist_lontano || '-';
    document.getElementById('v-dist-vicino').innerText = r.dist_vicino || '-';
    document.getElementById('view-ricetta-modal').classList.remove('hidden');
}

function closeViewRicettaModal() { document.getElementById('view-ricetta-modal').classList.add('hidden'); }

function deleteClient(id) {
    if (confirm("Sei sicuro di voler eliminare questo cliente?")) {
        appData.clients = appData.clients.filter(c => c.id !== id);
        saveData();
        initConformita();
        showToast("Cliente eliminato.");
    }
}

function refreshClientsTable() {
    const tbody = document.getElementById('clients-table-body');
    const azIndex = document.getElementById('az-index');
    const searchVal = document.getElementById('search-client').value.toLowerCase();

    tbody.innerHTML = ''; azIndex.innerHTML = '';

    let filtered = appData.clients.filter(c =>
        (c.nome && c.nome.toLowerCase().includes(searchVal)) ||
        (c.cf && c.cf.toLowerCase().includes(searchVal)) ||
        (c.piva && c.piva.toLowerCase().includes(searchVal))
    ).sort((a, b) => a.nome.localeCompare(b.nome));

    let currentLetter = ''; let lettersPresent = new Set();
    filtered.forEach(c => {
        let primaLettera = c.nome ? c.nome.charAt(0).toUpperCase() : '?';
        if (!primaLettera.match(/[A-Z]/i)) primaLettera = '#';
        if (primaLettera !== currentLetter) {
            currentLetter = primaLettera; lettersPresent.add(currentLetter);
            const trHeader = document.createElement('tr');
            trHeader.className = 'letter-header-row';
            trHeader.innerHTML = `<td colspan="4" class="letter-header" id="letter-${currentLetter}">${currentLetter}</td>`;
            tbody.appendChild(trHeader);
        }

        const iden = c.tipo === 'azienda' ? c.piva : c.cf;
        const tr = document.createElement('tr');

        let actions = `<button class="btn-icon info" onclick="viewRicetta('${c.id}')" title="Visualizza Diottrie"><i class="fas fa-eye"></i></button>`;
        if (c.telefono) { actions += `<button class="btn-icon whatsapp" onclick="openWhatsApp('${c.telefono}')" title="Invia WhatsApp"><i class="fab fa-whatsapp"></i></button>`; }
        actions += `<button class="btn-icon" onclick="openClientModal('${c.id}')" title="Modifica Cliente"><i class="fas fa-edit"></i></button>
                    <button class="btn-icon danger" onclick="deleteClient('${c.id}')" title="Elimina"><i class="fas fa-trash"></i></button>`;

        tr.innerHTML = `
            <td data-label="Nome Cliente"><strong>${c.nome}</strong></td>
            <td data-label="C.F. / P.IVA">${iden || '-'}</td>
            <td data-label="Città">${c.comune || '-'}</td>
            <td data-label="Azioni">${actions}</td>
        `;
        tbody.appendChild(tr);
    });

    const alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ#".split('');
    alphabet.forEach(letter => {
        if (lettersPresent.has(letter)) { azIndex.innerHTML += `<a onclick="scrollToLetter('${letter}')">${letter}</a>`; }
        else { azIndex.innerHTML += `<span style="color: gray; opacity:0.5; padding: 2px 5px;">${letter}</span>`; }
    });
}

function scrollToLetter(letter) {
    const el = document.getElementById(`letter-${letter}`);
    if (el) el.scrollIntoView({ behavior: 'smooth', block: 'center' });
}

function refreshClientsDropdown() {
    const selettore = document.getElementById('select-cliente');
    selettore.innerHTML = '<option value="">-- Seleziona o inserisci manualmente --</option>';
    const sortedClients = [...appData.clients].sort((a, b) => a.nome.localeCompare(b.nome));
    sortedClients.forEach(c => { selettore.innerHTML += `<option value="${c.id}">${c.nome}</option>`; });
}

/* ======================================================= */
/* FATTURE                                                   */
/* ======================================================= */
function initInvoiceTab() {
    const selInv = document.getElementById('invoice-inventory-select');
    if (selInv) {
        selInv.innerHTML = '<option value="">-- Seleziona articolo dal Magazzino --</option>';
        if (!appData.inventory) appData.inventory = [];
        appData.inventory.forEach(i => {
            const desc = `${i.marca} ${i.modello} - €${i.prezzo}`;
            selInv.innerHTML += `<option value="${i.id}">${desc}</option>`;
        });
    }
}

function loadClientIntoInvoice() {
    const id = document.getElementById('select-cliente').value;
    if (!id) {
        document.getElementById('input-nome').value = ''; document.getElementById('input-via').value = '';
        document.getElementById('input-comune').value = ''; document.getElementById('input-cap').value = '';
        document.getElementById('input-prov').value = ''; document.getElementById('input-cf').value = '';
        document.getElementById('input-piva').value = ''; document.getElementById('input-sdi').value = '';
    } else {
        const c = appData.clients.find(c => c.id === id);
        document.getElementById('tipo-' + c.tipo).checked = true;
        toggleClientFields();
        document.getElementById('input-nome').value = c.nome; document.getElementById('input-via').value = c.via;
        document.getElementById('input-comune').value = c.comune; document.getElementById('input-cap').value = c.cap;
        document.getElementById('input-prov').value = c.prov; document.getElementById('input-cf').value = c.cf;
        document.getElementById('input-piva').value = c.piva; document.getElementById('input-sdi').value = c.sdi || '';
    }
    updatePreview();
}

function toggleClientFields() {
    const isAzienda = document.getElementById('tipo-azienda').checked;
    document.getElementById('group-cf').classList.remove('hidden');
    document.getElementById('group-piva').classList.toggle('hidden', !isAzienda);
    document.getElementById('group-sdi').classList.toggle('hidden', !isAzienda);
    document.getElementById('prev-cf').style.display = 'block';
    document.getElementById('prev-piva').style.display = isAzienda ? 'block' : 'none';
    document.getElementById('prev-sdi').style.display = isAzienda ? 'block' : 'none';
    updatePreview();
}

function updatePreview() {
    const isAzienda = document.getElementById('tipo-azienda').checked;
    const via = document.getElementById('input-via').value;
    const cap = document.getElementById('input-cap').value;
    const comune = document.getElementById('input-comune').value;
    const prov = document.getElementById('input-prov').value.toUpperCase();

    let indirizzoCompleto = via;
    if (cap || comune || prov) { indirizzoCompleto += `\n${cap} ${comune} ${prov ? '(' + prov + ')' : ''}`; }

    document.getElementById('prev-nome').innerText = document.getElementById('input-nome').value || 'Nome Cliente';
    document.getElementById('prev-indirizzo').innerText = indirizzoCompleto || 'Indirizzo Cliente';

    const cfInput = document.getElementById('input-cf');
    cfInput.value = cfInput.value.toUpperCase();
    document.getElementById('prev-cf').innerText = 'C.F.: ' + cfInput.value;
    document.getElementById('prev-piva').innerText = 'P.IVA: ' + document.getElementById('input-piva').value;
    document.getElementById('prev-sdi').innerText = 'Cod. Destinatario: ' + document.getElementById('input-sdi').value;

    document.getElementById('prev-numero').innerText = document.getElementById('input-numero').value;
    const dataInput = document.getElementById('input-data').value;
    if (dataInput) { document.getElementById('prev-data').innerText = new Date(dataInput).toLocaleDateString('it-IT'); }
    document.getElementById('prev-pagamento').innerText = document.getElementById('input-pagamento').value;
}

function loadItemIntoInvoice() {
    const id = document.getElementById('invoice-inventory-select').value;
    if (!id) return;
    const item = appData.inventory.find(i => i.id === id);
    if (item) {
        document.getElementById('input-desc').value = `${item.categoria.toUpperCase()} - ${item.fornitore} ${item.marca} ${item.modello}`;
        document.getElementById('input-prezzo').value = item.prezzo;
    }
}

function addItem() {
    const desc = document.getElementById('input-desc').value;
    const subtext = document.getElementById('input-subtext').value;
    const priceStr = document.getElementById('input-prezzo').value;
    if (!desc.trim()) { showToast('Inserisci almeno la descrizione.', "error"); return; }

    const price = priceStr !== '' ? parseFloat(priceStr) : 0;
    invoiceItems.push({ desc: desc, subtext: subtext, price: price });

    document.getElementById('input-desc').value = '';
    document.getElementById('input-subtext').value = '';
    document.getElementById('input-prezzo').value = '';
    renderItems();
}

function removeItem(index) { invoiceItems.splice(index, 1); renderItems(); }

function renderItems() {
    const tbody = document.getElementById('invoice-items');
    tbody.innerHTML = ''; let totalAmount = 0;
    invoiceItems.forEach((item, index) => {
        const tr = document.createElement('tr');
        const priceText = item.price !== null ? item.price.toFixed(2) : '0.00';
        totalAmount += item.price;
        let descHtml = `<div>${item.desc}</div>`;
        if (item.subtext) descHtml += `<div style="font-size:12px; color:#666;">${item.subtext}</div>`;
        tr.innerHTML = `<td>${descHtml}</td><td class="text-right">${priceText}</td>
            <td data-html2canvas-ignore="true" class="text-right"><button style="background:none; border:none; color:red; cursor:pointer;" onclick="removeItem(${index})"><i class="fas fa-times"></i></button></td>`;
        tbody.appendChild(tr);
    });
    document.getElementById('prev-imponibile').innerText = totalAmount.toFixed(2);
    document.getElementById('prev-totale').innerText = totalAmount.toFixed(2);
}

function autoSaveClientFromInvoice() {
    const isAzienda = document.getElementById('tipo-azienda').checked;
    const nome = document.getElementById('input-nome').value.trim();
    const cf = document.getElementById('input-cf').value.toUpperCase().trim();
    const piva = document.getElementById('input-piva').value.trim();
    if (!nome) return;

    let exists = appData.clients.find(c => {
        if (isAzienda && piva && c.piva === piva) return true;
        if (!isAzienda && cf && c.cf === cf) return true;
        if (c.nome.toLowerCase() === nome.toLowerCase()) return true;
        return false;
    });

    if (!exists) {
        appData.clients.push({
            id: Date.now().toString() + "-auto", nome: nome, tipo: isAzienda ? 'azienda' : 'fisica',
            cf: cf, piva: piva, sdi: document.getElementById('input-sdi').value.trim(),
            via: document.getElementById('input-via').value.trim(), comune: document.getElementById('input-comune').value.trim(),
            cap: document.getElementById('input-cap').value.trim(), prov: document.getElementById('input-prov').value.toUpperCase().trim(),
            telefono: '', note: 'Salvato in automatico dalla fattura',
            ricetta: { dist_lontano: '', dist_vicino: '', dx_sf_l: '', dx_cil_l: '', dx_ax_l: '', sx_sf_l: '', sx_cil_l: '', sx_ax_l: '', dx_sf_i: '', dx_cil_i: '', dx_ax_i: '', sx_sf_i: '', sx_cil_i: '', sx_ax_i: '', dx_sf_v: '', dx_cil_v: '', dx_ax_v: '', sx_sf_v: '', sx_cil_v: '', sx_ax_v: '' }
        });
        initConformita();
    }
}

function generateAndSaveInvoice() {
    let clientName = document.getElementById('input-nome').value.trim();
    if (!clientName) { showToast("Inserisci il nome del cliente!", "error"); return; }
    if (invoiceItems.length === 0) { showToast("Aggiungi almeno una voce!", "error"); return; }

    const numFattura = parseInt(document.getElementById('input-numero').value);
    const dataFattura = document.getElementById('input-data').value;
    const totaleFattura = parseFloat(document.getElementById('prev-totale').innerText);

    autoSaveClientFromInvoice();

    appData.invoices.push({
        id: Date.now().toString(),
        numero: numFattura,
        data: dataFattura,
        cliente: clientName,
        metodo: document.getElementById('input-pagamento').value,
        totale: totaleFattura
    });

    saveData();

    document.getElementById('mobile-preview-container').classList.remove('show-mobile');
    const element = document.getElementById('invoice-preview');
    element.style.boxShadow = 'none';

    const opt = {
        margin: 0, filename: `Fattura_${numFattura}_${new Date(dataFattura).getFullYear()}_${clientName}.pdf`,
        image: { type: 'jpeg', quality: 0.98 },
        html2canvas: { scale: 2, scrollY: 0 },
        jsPDF: { unit: 'mm', format: 'a4', orientation: 'portrait' },
        pagebreak: { mode: 'avoid-all' }
    };

    html2pdf().set(opt).from(element).save().then(() => {
        element.style.boxShadow = 'var(--shadow)';
        showToast("Fattura scaricata PDF e salvata!");
        document.getElementById('input-numero').value = calculateNextInvoiceNumber(document.getElementById('input-data').value);
        invoiceItems = []; renderItems();
    });
}

function refreshInvoicesTable() {
    const tbody = document.getElementById('invoices-table-body');
    tbody.innerHTML = '';
    let filteredRevenue = 0;

    const searchVal = (document.getElementById('filter-inv-search')?.value || '').toLowerCase();
    const yearVal = document.getElementById('filter-inv-year')?.value;
    const monthVal = document.getElementById('filter-inv-month')?.value;

    currentFilteredInvoices = appData.invoices.filter(i => {
        const d = new Date(i.data);
        const iYear = d.getFullYear().toString();
        const iMonth = d.getMonth().toString();

        const matchSearch = i.cliente.toLowerCase().includes(searchVal) || i.numero.toString().includes(searchVal);
        const matchYear = yearVal === "" || iYear === yearVal;
        const matchMonth = monthVal === "" || iMonth === monthVal;

        return matchSearch && matchYear && matchMonth;
    });

    currentFilteredInvoices.sort((a, b) => new Date(b.data) - new Date(a.data));

    currentFilteredInvoices.forEach(inv => {
        filteredRevenue += inv.totale;
        const tr = document.createElement('tr');
        const d = new Date(inv.data);
        const dateStr = d.toLocaleDateString('it-IT');

        tr.innerHTML = `
            <td data-label="N° e Anno"><strong>#${inv.numero}/${d.getFullYear()}</strong></td>
            <td data-label="Data Emissione">${dateStr}</td>
            <td data-label="Cliente">${inv.cliente}</td>
            <td data-label="Metodo Pag.">${inv.metodo}</td>
            <td data-label="Importo"><strong>€ ${inv.totale.toFixed(2)}</strong></td>
            <td data-label="Azioni">
                <button class="btn-icon danger" onclick="deleteInvoice('${inv.id}')" title="Elimina dallo storico"><i class="fas fa-trash"></i></button>
            </td>
        `;
        tbody.appendChild(tr);
    });

    document.getElementById('total-revenue').innerText = `€ ${filteredRevenue.toFixed(2)}`;

    if (searchVal || yearVal || monthVal) {
        document.getElementById('label-totale-filtrato').innerText = "Fatturato nel Periodo Filtrato";
    } else {
        document.getElementById('label-totale-filtrato').innerText = "Fatturato Globale (Tutto lo Storico)";
    }

    updateChart();
}

function exportCSV() {
    if (currentFilteredInvoices.length === 0) { showToast("Nessuna fattura trovata con questi filtri", "error"); return; }

    let csvContent = "data:text/csv;charset=utf-8,";
    csvContent += "Numero,Anno,Data,Cliente,Metodo Pagamento,Totale Euro\n";

    currentFilteredInvoices.forEach(i => {
        let d = new Date(i.data);
        let dateStr = d.toLocaleDateString('it-IT');
        let safeName = i.cliente.replace(/,/g, " ");
        let row = `${i.numero},${d.getFullYear()},${dateStr},${safeName},${i.metodo},${i.totale.toFixed(2)}`;
        csvContent += row + "\r\n";
    });

    var encodedUri = encodeURI(csvContent);
    var link = document.createElement("a");
    link.setAttribute("href", encodedUri);
    link.setAttribute("download", `Fatture_Export_${new Date().getTime()}.csv`);
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    showToast("Documento Excel scaricato!");
}

function deleteInvoice(id) {
    if (confirm("Vuoi eliminare questa fattura? (Il numero non verrà riassegnato automaticamente)")) {
        appData.invoices = appData.invoices.filter(i => i.id !== id);
        saveData();
        showToast("Fattura eliminata.");
    }
}

function updateChart() {
    const ctx = document.getElementById('revenueChart');
    if (!ctx) return;

    const mode = document.getElementById('chart-mode').value;
    const selectedYear = parseInt(document.getElementById('chart-year').value) || new Date().getFullYear();

    document.getElementById('chart-year').style.display = mode === 'monthly' ? 'block' : 'none';

    let labels = [];
    let chartData = [];
    let labelTitle = "";

    if (mode === 'monthly') {
        const monthlyData = new Array(12).fill(0);
        appData.invoices.forEach(inv => {
            const d = new Date(inv.data);
            if (d.getFullYear() === selectedYear) { monthlyData[d.getMonth()] += inv.totale; }
        });
        labels = ['Gen', 'Feb', 'Mar', 'Apr', 'Mag', 'Giu', 'Lug', 'Ago', 'Set', 'Ott', 'Nov', 'Dic'];
        chartData = monthlyData;
        labelTitle = `Fatturato ${selectedYear} (€)`;
    } else {
        const yearlyMap = {};
        appData.invoices.forEach(inv => {
            const y = new Date(inv.data).getFullYear();
            yearlyMap[y] = (yearlyMap[y] || 0) + inv.totale;
        });

        labels = Object.keys(yearlyMap).sort();
        if (labels.length === 0) labels = [new Date().getFullYear().toString()];
        labels.forEach(y => chartData.push(yearlyMap[y] || 0));
        labelTitle = "Confronto Annuale Globale (€)";
    }

    const isDark = document.documentElement.getAttribute('data-theme') === 'dark';
    Chart.defaults.color = isDark ? '#e0e0e0' : '#666';

    if (revenueChartInstance) revenueChartInstance.destroy();

    revenueChartInstance = new Chart(ctx, {
        type: 'bar',
        data: {
            labels: labels,
            datasets: [{
                label: labelTitle,
                data: chartData,
                backgroundColor: '#3498db',
                borderRadius: 6
            }]
        },
        options: {
            responsive: true, maintainAspectRatio: false,
            plugins: { legend: { display: true } },
            scales: {
                y: { beginAtZero: true, grid: { color: isDark ? '#333' : '#eee' } },
                x: { grid: { color: isDark ? '#333' : '#eee' } }
            }
        }
    });
}

/* ======================================================= */
/* MAGAZZINO & INVENTARIO                                    */
/* ======================================================= */
function toggleInventoryFields() {
    const cat = document.getElementById('inv-categoria').value;
    document.getElementById('inv-frame-details').classList.toggle('hidden', cat !== 'montatura');
    document.getElementById('inv-lens-details').classList.toggle('hidden', cat !== 'lente');
}

function saveInventoryItem(e) {
    e.preventDefault();
    if (!appData.inventory) appData.inventory = [];

    const id = document.getElementById('inv-id').value || Date.now().toString();
    const item = {
        id: id,
        categoria: document.getElementById('inv-categoria').value,
        fornitore: document.getElementById('inv-fornitore').value,
        marca: document.getElementById('inv-marca').value,
        modello: document.getElementById('inv-modello').value,
        prezzo: parseFloat(document.getElementById('inv-prezzo').value) || 0,
        qty: parseInt(document.getElementById('inv-qty').value) || 0,

        calibro: document.getElementById('inv-calibro').value,
        ponte: document.getElementById('inv-ponte').value,
        aste: document.getElementById('inv-aste').value,
        colore: document.getElementById('inv-colore').value,

        mat: document.getElementById('inv-mat').value,
        trat: document.getElementById('inv-trat').value,
        dia: document.getElementById('inv-dia').value,
        col_perc: document.getElementById('inv-col-perc').value,
        filtro: document.getElementById('inv-filtro').value
    };

    const index = appData.inventory.findIndex(i => i.id === id);
    if (index > -1) appData.inventory[index] = item;
    else appData.inventory.push(item);

    saveData();
    resetInventoryForm();
    showToast("Articolo salvato in magazzino!");
}

function resetInventoryForm() {
    document.getElementById('inventory-form').reset();
    document.getElementById('inv-id').value = '';
    document.getElementById('inv-form-title').innerText = 'Aggiungi Articolo';
    toggleInventoryFields();
}

function refreshInventoryTable() {
    const tbody = document.getElementById('inventory-table-body');
    if (!tbody) return;
    tbody.innerHTML = '';

    if (!appData.inventory) appData.inventory = [];
    const searchVal = (document.getElementById('search-inventory')?.value || '').toLowerCase();

    let filtered = appData.inventory.filter(i =>
        (i.marca && i.marca.toLowerCase().includes(searchVal)) ||
        (i.fornitore && i.fornitore.toLowerCase().includes(searchVal)) ||
        (i.modello && i.modello.toLowerCase().includes(searchVal))
    );

    filtered.sort((a, b) => a.marca.localeCompare(b.marca));

    filtered.forEach(i => {
        const tr = document.createElement('tr');
        const isFrame = i.categoria === 'montatura';
        let subDetails = isFrame ? `<br><small style="color:#777;">Cal: ${i.calibro} | Ponte: ${i.ponte} | Col: ${i.colore}</small>` : '';
        if (i.categoria === 'lente') subDetails = `<br><small style="color:#777;">Mat: ${i.mat} | Ø: ${i.dia}</small>`;

        tr.innerHTML = `
            <td data-label="Categoria" style="text-transform: capitalize;">${i.categoria}</td>
            <td data-label="Fornitore">${i.fornitore}</td>
            <td data-label="Marca & Modello"><strong>${i.marca} ${i.modello}</strong>${subDetails}</td>
            <td data-label="Giacenza">${i.qty}</td>
            <td data-label="Prezzo (€)">€ ${i.prezzo.toFixed(2)}</td>
            <td data-label="Azioni">
                <button class="btn-icon" onclick="editInventoryItem('${i.id}')" title="Modifica"><i class="fas fa-edit"></i></button>
                <button class="btn-icon danger" onclick="deleteInventoryItem('${i.id}')" title="Elimina"><i class="fas fa-trash"></i></button>
            </td>
        `;
        tbody.appendChild(tr);
    });

    initInvoiceTab();
    initConformita();
}

function editInventoryItem(id) {
    const item = appData.inventory.find(i => i.id === id);
    if (item) {
        document.getElementById('inv-form-title').innerText = 'Modifica Articolo';
        document.getElementById('inv-id').value = item.id;
        document.getElementById('inv-categoria').value = item.categoria;
        toggleInventoryFields();
        document.getElementById('inv-fornitore').value = item.fornitore || '';
        document.getElementById('inv-marca').value = item.marca || '';
        document.getElementById('inv-modello').value = item.modello || '';
        document.getElementById('inv-prezzo').value = item.prezzo || 0;
        document.getElementById('inv-qty').value = item.qty || 1;

        document.getElementById('inv-calibro').value = item.calibro || '';
        document.getElementById('inv-ponte').value = item.ponte || '';
        document.getElementById('inv-aste').value = item.aste || '';
        document.getElementById('inv-colore').value = item.colore || '';

        document.getElementById('inv-mat').value = item.mat || '';
        document.getElementById('inv-trat').value = item.trat || '';
        document.getElementById('inv-dia').value = item.dia || '';
        document.getElementById('inv-col-perc').value = item.col_perc || '';
        document.getElementById('inv-filtro').value = item.filtro || '';
    }
}

function deleteInventoryItem(id) {
    if (confirm("Sei sicuro di voler eliminare questo articolo dal magazzino?")) {
        appData.inventory = appData.inventory.filter(i => i.id !== id);
        saveData();
        showToast("Articolo eliminato.");
    }
}


/* ======================================================= */
/* LOGICA DICHIARAZIONE DI CONFORMITÀ                      */
/* ======================================================= */

function initConformita() {
    const selectCliente = document.getElementById('conf-select-cliente');
    if (selectCliente) {
        selectCliente.innerHTML = '<option value="">-- Compila Manualmente --</option>';
        appData.clients.sort((a, b) => a.nome.localeCompare(b.nome)).forEach(c => {
            selectCliente.innerHTML += `<option value="${c.id}">${c.nome}</option>`;
        });
    }

    const selFrame = document.getElementById('conf-inventory-frame');
    const selLens = document.getElementById('conf-inventory-lens');
    if (!appData.inventory) appData.inventory = [];

    if (selFrame) {
        selFrame.innerHTML = '<option value="">-- Seleziona Montatura --</option>';
        appData.inventory.filter(i => i.categoria === 'montatura').forEach(i => {
            selFrame.innerHTML += `<option value="${i.id}">${i.marca} ${i.modello} (${i.colore})</option>`;
        });
    }
    if (selLens) {
        selLens.innerHTML = '<option value="">-- Seleziona Lente --</option>';
        appData.inventory.filter(i => i.categoria === 'lente').forEach(i => {
            selLens.innerHTML += `<option value="${i.id}">${i.marca} ${i.modello} - Ø${i.dia}</option>`;
        });
    }

    const confData = document.getElementById('conf-data');
    if (confData) confData.valueAsDate = new Date();

    applyBranding();
    updateConformityPreview();
}

function loadFrameIntoConformity() {
    const id = document.getElementById('conf-inventory-frame').value;
    if (!id) return;
    const i = appData.inventory.find(item => item.id === id);
    if (i) {
        const setVal = (id, val) => { const el = document.getElementById(id); if (el) el.value = val || ''; };
        setVal('conf-mont-fornitore', i.fornitore);
        setVal('conf-mont-marca', i.marca);
        setVal('conf-mont-modello', i.modello);
        setVal('conf-mont-calibro', i.calibro);
        setVal('conf-mont-ponte', i.ponte);
        setVal('conf-mont-aste', i.aste);
        setVal('conf-mont-colore', i.colore);
        updateConformityPreview();
    }
}

function loadLensIntoConformity() {
    const id = document.getElementById('conf-inventory-lens').value;
    if (!id) return;
    const i = appData.inventory.find(item => item.id === id);
    if (i) {
        const setVal = (id, val) => { const el = document.getElementById(id); if (el) el.value = val || ''; };
        setVal('conf-lenti-fornitore', i.fornitore);
        setVal('conf-lenti-tipo', i.marca + ' ' + i.modello);
        setVal('conf-lenti-mat', i.mat);
        setVal('conf-lenti-trat', i.trat);
        setVal('conf-lenti-dia', i.dia);
        setVal('conf-lenti-col', i.col_perc);
        setVal('conf-lenti-filtro', i.filtro);
        updateConformityPreview();
    }
}

function loadClientIntoConformity() {
    const clientId = document.getElementById('conf-select-cliente').value;
    const setVal = (id, val) => { const el = document.getElementById(id); if (el) el.value = val || ''; };

    if (clientId) {
        const c = appData.clients.find(x => x.id == clientId);
        if (c) {
            setVal('conf-manual-nome', `${c.nome || ''} ${c.cognome || ''}`.trim());
            setVal('conf-manual-via', c.indirizzo || c.via || '');
            setVal('conf-manual-citta', c.citta || c.comune || '');
            setVal('conf-manual-prov', c.provincia || c.prov || '');
            setVal('conf-manual-cf', c.codiceFiscale || c.cf || '');
            setVal('conf-manual-tel', c.telefono || '');

            if (c.ricetta) {
                const r = c.ricetta;
                setVal('rx-dx-sf-l', r.dx_sf_l); setVal('rx-dx-cil-l', r.dx_cil_l); setVal('rx-dx-ax-l', r.dx_ax_l); setVal('rx-dx-pr-l', r.dx_pr_l); setVal('rx-dx-ba-l', r.dx_ba_l);
                setVal('rx-dx-sf-v', r.dx_sf_v); setVal('rx-dx-cil-v', r.dx_cil_v); setVal('rx-dx-ax-v', r.dx_ax_v); setVal('rx-dx-pr-v', r.dx_pr_v); setVal('rx-dx-ba-v', r.dx_ba_v);
                setVal('rx-sx-sf-l', r.sx_sf_l); setVal('rx-sx-cil-l', r.sx_cil_l); setVal('rx-sx-ax-l', r.sx_ax_l); setVal('rx-sx-pr-l', r.sx_pr_l); setVal('rx-sx-ba-l', r.sx_ba_l);
                setVal('rx-sx-sf-v', r.sx_sf_v); setVal('rx-sx-cil-v', r.sx_cil_v); setVal('rx-sx-ax-v', r.sx_ax_v); setVal('rx-sx-pr-v', r.sx_pr_v); setVal('rx-sx-ba-v', r.sx_ba_v);
                setVal('conf-dil', r.dist_lontano);
                setVal('conf-div', r.dist_vicino);
            }
        }
    } else {
        ['nome', 'via', 'citta', 'prov', 'cf', 'tel'].forEach(id => setVal(`conf-manual-${id}`, ''));
        const rIds = ['dx-sf-l', 'dx-cil-l', 'dx-ax-l', 'dx-pr-l', 'dx-ba-l', 'dx-sf-v', 'dx-cil-v', 'dx-ax-v', 'dx-pr-v', 'dx-ba-v',
            'sx-sf-l', 'sx-cil-l', 'sx-ax-l', 'sx-pr-l', 'sx-ba-l', 'sx-sf-v', 'sx-cil-v', 'sx-ax-v', 'sx-pr-v', 'sx-ba-v'];
        rIds.forEach(id => setVal(`rx-${id}`, ''));
        setVal('conf-dil', ''); setVal('conf-div', '');
    }
    updateConformityPreview();
}

function updateConformityPreview() {
    const sync = (inputId, prevId) => {
        const input = document.getElementById(inputId);
        const prev = document.getElementById(prevId);
        if (input && prev) prev.textContent = input.value;
    };

    // Client
    sync('conf-manual-nome', 'prev-conf-nome'); sync('conf-manual-via', 'prev-conf-via'); sync('conf-manual-citta', 'prev-conf-citta');
    sync('conf-manual-prov', 'prev-conf-prov'); sync('conf-manual-cf', 'prev-conf-cf'); sync('conf-manual-tel', 'prev-conf-tel');

    // Montatura
    sync('conf-mont-fornitore', 'prev-mont-forn'); sync('conf-mont-marca', 'prev-mont-marca'); sync('conf-mont-modello', 'prev-mont-modello');
    sync('conf-mont-calibro', 'prev-mont-calibro'); sync('conf-mont-ponte', 'prev-mont-ponte'); sync('conf-mont-aste', 'prev-mont-aste'); sync('conf-mont-colore', 'prev-mont-colore');

    // Lenti
    sync('conf-lenti-fornitore', 'prev-lenti-forn'); sync('conf-lenti-tipo', 'prev-lenti-tipo'); sync('conf-lenti-mat', 'prev-lenti-mat');
    sync('conf-lenti-trat', 'prev-lenti-trat'); sync('conf-lenti-dia', 'prev-lenti-dia'); sync('conf-lenti-col', 'prev-lenti-col'); sync('conf-lenti-filtro', 'prev-lenti-filtro');

    // Optometria
    sync('rx-dx-sf-l', 'prev-rx-dx-sf-l'); sync('rx-dx-cil-l', 'prev-rx-dx-cil-l'); sync('rx-dx-ax-l', 'prev-rx-dx-ax-l'); sync('rx-dx-pr-l', 'prev-rx-dx-pr-l'); sync('rx-dx-ba-l', 'prev-rx-dx-ba-l');
    sync('rx-dx-sf-v', 'prev-rx-dx-sf-v'); sync('rx-dx-cil-v', 'prev-rx-dx-cil-v'); sync('rx-dx-ax-v', 'prev-rx-dx-ax-v'); sync('rx-dx-pr-v', 'prev-rx-dx-pr-v'); sync('rx-dx-ba-v', 'prev-rx-dx-ba-v');
    sync('rx-sx-sf-l', 'prev-rx-sx-sf-l'); sync('rx-sx-cil-l', 'prev-rx-sx-cil-l'); sync('rx-sx-ax-l', 'prev-rx-sx-ax-l'); sync('rx-sx-pr-l', 'prev-rx-sx-pr-l'); sync('rx-sx-ba-l', 'prev-rx-sx-ba-l');
    sync('rx-sx-sf-v', 'prev-rx-sx-sf-v'); sync('rx-sx-cil-v', 'prev-rx-sx-cil-v'); sync('rx-sx-ax-v', 'prev-rx-sx-ax-v'); sync('rx-sx-pr-v', 'prev-rx-sx-pr-v'); sync('rx-sx-ba-v', 'prev-rx-sx-ba-v');
    sync('conf-dil', 'prev-conf-dil'); sync('conf-div', 'prev-conf-div');
    sync('conf-prescrittore', 'prev-conf-prescrittore');

    const valLentiCE = document.getElementById('conf-lenti-ce') ? document.getElementById('conf-lenti-ce').value : 'si';
    document.getElementById('prev-lenti-ce-si').className = valLentiCE === 'si' ? 'ce-box active' : 'ce-box';
    document.getElementById('prev-lenti-ce-no').className = valLentiCE === 'no' ? 'ce-box active' : 'ce-box';

    const valMontCE = document.getElementById('conf-mont-ce') ? document.getElementById('conf-mont-ce').value : 'si';
    document.getElementById('prev-mont-ce-si').className = valMontCE === 'si' ? 'ce-box active' : 'ce-box';
    document.getElementById('prev-mont-ce-no').className = valMontCE === 'no' ? 'ce-box active' : 'ce-box';

    const valAsse = document.getElementById('conf-sistema-asse') ? document.getElementById('conf-sistema-asse').value : 'tabo';
    document.getElementById('prev-asse-tabo').textContent = valAsse === 'tabo' ? 'X' : '\u00A0';
    document.getElementById('prev-asse-int').textContent = valAsse === 'internazionale' ? 'X' : '\u00A0';

    const dataInput = document.getElementById('conf-data');
    if (dataInput && dataInput.value) {
        const dataFormattata = new Date(dataInput.value).toLocaleDateString('it-IT');
        document.getElementById('prev-conf-data-firma').textContent = dataFormattata;
        document.getElementById('prev-conf-data-footer').textContent = dataFormattata;
    }
}

// Funzione globale per scaricare la dichiarazione di conformità in PDF
window.generateConformityPDF = function () {
    const elNome = document.getElementById('conf-manual-nome');
    if (!elNome || !elNome.value) { showToast('Inserisci il nome del cliente!', 'error'); return; }

    const element = document.getElementById('conformity-preview');
    const opt = {
        margin: 0,
        filename: `Dichiarazione_Conformita_${elNome.value.replace(/ /g, '_')}.pdf`,
        image: { type: 'jpeg', quality: 0.98 },
        html2canvas: { scale: 2, scrollY: 0 },
        jsPDF: { unit: 'mm', format: 'a4', orientation: 'portrait' }
    };

    document.getElementById('conformity-mobile-preview-container')?.classList.remove('show-mobile');
    element.style.boxShadow = 'none';

    html2pdf().set(opt).from(element).save().then(() => {
        element.style.boxShadow = '0 5px 20px rgba(0,0,0,0.15)';
        showToast('Dichiarazione scaricata in PDF!');
    });
};
