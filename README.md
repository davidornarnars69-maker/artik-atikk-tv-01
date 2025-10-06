[app.js](https://github.com/user-attachments/files/22723187/app.js)

// ====== CONFIG ======
// The publishable key is injected from the server at /config
let stripe = null;
let publishableKey = null;

const state = {
  admin: false,
  products: [],
  activeSize: {},
}

// Load initial data
window.addEventListener('DOMContentLoaded', async () => {
  await fetchConfig();
  await loadProducts();
  loadStoreStatus();
  wireAdmin();
});

async function fetchConfig() {
  try {
    const res = await fetch('/config');
    if (!res.ok) throw new Error('No config');
    const cfg = await res.json();
    publishableKey = cfg.publishableKey;
    stripe = Stripe(publishableKey);
  } catch (e) {
    console.warn('Stripe config not available. Checkout will be disabled until server is running.');
  }
}

async function loadProducts() {
  // Check for locally saved admin products
  const local = localStorage.getItem('artik_products_override');
  if (local) {
    try {
      state.products = JSON.parse(local);
    } catch { /* ignore */ }
  }
  if (!state.products.length) {
    try {
      const res = await fetch('/api/products');
      state.products = await res.json();
    } catch (e) {
      console.warn('Falling back to embedded products list.');
      state.products = [{
        id: 'hoodie-black',
        name: 'ARTIK ATIKK Hoodie',
        price: 9990,
        currency: 'ISK',
        sizes: ['S','M','L','XL','XXL'],
        image: '/images/hoodie1.jpg',
        gallery: ['/images/hoodie1.jpg','/images/hoodie2.jpg'],
        description: 'Heavyweight black hoodie with ARTIK ATIKK Reykjavik print.'
      }];
    }
  }
  renderProducts();
}

function renderProducts() {
  const root = document.getElementById('products');
  root.innerHTML = '';
  state.products.forEach(p => {
    const card = document.createElement('div');
    card.className = 'card';
    card.innerHTML = `
      <img src="${p.image}" alt="${p.name}">
      <div class="body">
        <h3>${p.name}</h3>
        <p class="small">${p.description || ''}</p>
        <div class="price">${p.price.toLocaleString()} ${p.currency || 'ISK'}</div>
        <div class="size-row" id="sizes-${p.id}">
          ${(p.sizes||[]).map(s => `<span class="tag" data-sz="${s}" data-id="${p.id}">${s}</span>`).join('')}
        </div>
        <div style="display:flex;gap:8px;margin-top:12px">
          <button class="btn btn-primary" data-buy="${p.id}">Buy</button>
          <button class="btn btn-ghost" data-qty-minus="${p.id}">-</button>
          <span id="qty-${p.id}" class="small">1</span>
          <button class="btn btn-ghost" data-qty-plus="${p.id}">+</button>
        </div>
      </div>
    `;
    root.appendChild(card);
  });

  // Set up size selection
  document.querySelectorAll('.size-row .tag').forEach(el => {
    el.addEventListener('click', () => {
      const id = el.dataset.id;
      state.activeSize[id] = el.dataset.sz;
      document.querySelectorAll(`#sizes-${id} .tag`).forEach(t => t.classList.remove('active'));
      el.classList.add('active');
    });
  });

  // Quantity buttons
  document.querySelectorAll('[data-qty-plus]').forEach(btn => {
    btn.addEventListener('click', () => {
      const id = btn.dataset.qtyPlus;
      const qtyEl = document.getElementById(`qty-${id}`);
      let q = parseInt(qtyEl.textContent, 10) || 1;
      qtyEl.textContent = String(Math.min(q+1, 10));
    });
  });
  document.querySelectorAll('[data-qty-minus]').forEach(btn => {
    btn.addEventListener('click', () => {
      const id = btn.dataset.qtyMinus;
      const qtyEl = document.getElementById(`qty-${id}`);
      let q = parseInt(qtyEl.textContent, 10) || 1;
      qtyEl.textContent = String(Math.max(q-1, 1));
    });
  });

  // Buy buttons
  document.querySelectorAll('[data-buy]').forEach(btn => {
    btn.addEventListener('click', async () => {
      if (getStoreClosed()) {
        alert('Store is closed.');
        return;
      }
      const id = btn.dataset.buy;
      const product = state.products.find(p => p.id === id);
      const size = state.activeSize[id] || (product.sizes ? product.sizes[0] : null);
      const qtyEl = document.getElementById(`qty-${id}`);
      const quantity = parseInt(qtyEl.textContent, 10) || 1;

      if (!stripe) {
        alert('Checkout backend not running. Start the server (node server.js).');
        return;
      }

      try {
        const res = await fetch('/create-checkout-session', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ items: [{ id, quantity, size }] })
        });
        const data = await res.json();
        if (data.error) throw new Error(data.error);
        const result = await stripe.redirectToCheckout({ sessionId: data.id });
        if (result.error) alert(result.error.message);
      } catch (e) {
        alert('Checkout error: ' + e.message);
      }
    });
  });
}

// ===== Store open/closed =====
function getStoreClosed() {
  return localStorage.getItem('artik_store_closed') === '1';
}
function setStoreClosed(closed) {
  localStorage.setItem('artik_store_closed', closed ? '1' : '0');
  document.getElementById('statusText').textContent = closed ? 'Closed' : 'Open';
  document.getElementById('closedOverlay').classList.toggle('hidden', !closed);
}
function loadStoreStatus() {
  setStoreClosed(getStoreClosed());
}

// ===== Admin =====
function wireAdmin() {
  const loginBtn = document.getElementById('loginBtn');
  const logoutBtn = document.getElementById('logoutBtn');
  const passInput = document.getElementById('adminPass');
  const adminControls = document.getElementById('adminControls');
  const adminLogin = document.getElementById('adminLogin');
  const storeToggle = document.getElementById('storeToggle');

  // restore
  if (localStorage.getItem('artik_admin') === '1') {
    state.admin = true;
    adminControls.classList.remove('hidden');
    adminLogin.classList.add('hidden');
  }

  loginBtn.addEventListener('click', () => {
    const pass = passInput.value.trim();
    const saved = localStorage.getItem('artik_admin_pw') || 'admin123';
    if (pass && pass === saved) {
      state.admin = true;
      localStorage.setItem('artik_admin', '1');
      adminControls.classList.remove('hidden');
      adminLogin.classList.add('hidden');
    } else {
      alert('Wrong password');
    }
  });

  logoutBtn.addEventListener('click', () => {
    state.admin = false;
    localStorage.removeItem('artik_admin');
    adminControls.classList.add('hidden');
    adminLogin.classList.remove('hidden');
  });

  storeToggle.checked = !getStoreClosed();
  storeToggle.addEventListener('change', () => {
    setStoreClosed(!storeToggle.checked);
  });

  document.getElementById('saveProduct').addEventListener('click', () => {
    const p = collectProductFromForm();
    if (!p) return;
    const idx = state.products.findIndex(x => x.id === p.id);
    if (idx >= 0) state.products[idx] = p; else state.products.push(p);
    localStorage.setItem('artik_products_override', JSON.stringify(state.products));
    renderProducts();
    alert('Saved locally. Export and replace server data/products.json to enable checkout for this product.');
  });

  document.getElementById('exportProducts').addEventListener('click', () => {
    const data = JSON.stringify(state.products, null, 2);
    const blob = new Blob([data], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'products.json';
    a.click();
    URL.revokeObjectURL(url);
  });
}

function collectProductFromForm() {
  const id = document.getElementById('p_id').value.trim();
  const name = document.getElementById('p_name').value.trim();
  const price = parseInt(document.getElementById('p_price').value, 10);
  const sizes = document.getElementById('p_sizes').value.split(',').map(s => s.trim()).filter(Boolean);
  const image = document.getElementById('p_image').value.trim();
  const gallery = document.getElementById('p_gallery').value.split(',').map(s => s.trim()).filter(Boolean);
  const description = document.getElementById('p_desc').value.trim();

  if (!id || !name || !price || !image) {
    alert('Please fill at least ID, Name, Price, and Image');
    return null;
  }
  return { id, name, price, currency: 'ISK', sizes, image, gallery, description };
}
