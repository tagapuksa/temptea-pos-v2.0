import React, { useState, useEffect } from 'react';

function App() {
  const [isLoggedIn, setIsLoggedIn] = useState(false);
  const [pinInput, setPinInput] = useState('');
  const MASTER_PIN = "1234";

  const [products, setProducts] = useState(() => {
    const saved = localStorage.getItem('teamptea_inventory');
    return saved ? JSON.parse(saved) : [
      { id: 1, name: 'Matcha Latte', price: 110, available: true, type: 'Drink', icon: '🍵', options: 'Pearls, Nata', stock: 50 },
      { id: 2, name: 'Classic Burger', price: 85, available: true, type: 'Food', icon: '🍔', options: 'Extra Cheese, Egg', stock: 20 }
    ];
  });

  const [sales, setSales] = useState(() => {
    const saved = localStorage.getItem('teamptea_sales');
    return saved ? JSON.parse(saved) : [];
  });

  const [receiptSettings, setReceiptSettings] = useState(() => {
    const saved = localStorage.getItem('teamptea_receipt');
    return saved ? JSON.parse(saved) : { logo: '🍋', footer: 'Thank you for choosing TEMPTEA!' };
  });

  const [currentMode, setCurrentMode] = useState('cashier'); 
  const [cart, setCart] = useState([]);
  const [newItem, setNewItem] = useState({ name: '', price: '', type: 'Drink', icon: '📦', options: '', stock: 0 });
  const [selectedProduct, setSelectedProduct] = useState(null);
  const [editingProduct, setEditingProduct] = useState(null);
  const [tempOptions, setTempOptions] = useState({ main: '100% Sugar', sub: 'None', isDiscounted: false });

  useEffect(() => {
    localStorage.setItem('teamptea_inventory', JSON.stringify(products));
    localStorage.setItem('teamptea_sales', JSON.stringify(sales));
    localStorage.setItem('teamptea_receipt', JSON.stringify(receiptSettings));
  }, [products, sales, receiptSettings]);

  const calculateCartTotal = () => {
    return cart.reduce((sum, item) => {
      const price = Number(item.price);
      return sum + (item.isDiscounted ? price * 0.8 : price);
    }, 0);
  };

  const getDailyTotal = () => {
    const today = new Date().toLocaleDateString();
    return sales
      .filter(sale => new Date(sale.date).toLocaleDateString() === today)
      .reduce((sum, sale) => sum + sale.total, 0);
  };

  const handleLogin = (e) => {
    e.preventDefault();
    if (pinInput === MASTER_PIN) { setIsLoggedIn(true); setPinInput(''); }
    else { alert("Wrong PIN"); setPinInput(''); }
  };

  const handlePrintAndComplete = () => {
    if (cart.length === 0) return;
    window.print();
    const updatedProducts = products.map(p => {
      const itemsInCart = cart.filter(item => item.id === p.id).length;
      return { ...p, stock: Math.max(0, p.stock - itemsInCart) };
    });
    setProducts(updatedProducts);
    const total = calculateCartTotal();
    const newSale = {
      id: Date.now(),
      date: new Date().toLocaleString(),
      items: cart.map(i => `${i.name}${i.isDiscounted ? ' (Disc)' : ''}`).join(' | '),
      total: total
    };
    setSales([...sales, newSale]);
    setCart([]);
  };

  const handleImageUpload = (e, target = 'new') => {
    const file = e.target.files[0];
    const reader = new FileReader();
    reader.onloadend = () => {
      if (target === 'edit') setEditingProduct({ ...editingProduct, icon: reader.result });
      else if (target === 'receipt') setReceiptSettings({ ...receiptSettings, logo: reader.result });
      else setNewItem({ ...newItem, icon: reader.result });
    };
    if (file) reader.readAsDataURL(file);
  };

  const exportToExcel = () => {
    let csvContent = "data:text/csv;charset=utf-8,ID,Date,Items,Total\n";
    sales.forEach(s => {
      csvContent += `${s.id},${s.date},"${s.items}",${s.total}\n`;
    });
    const encodedUri = encodeURI(csvContent);
    const link = document.createElement("a");
    link.setAttribute("href", encodedUri);
    link.setAttribute("download", "temptea_sales_report.csv");
    document.body.appendChild(link);
    link.click();
  };

  const renderIcon = (p) => {
    if (!p.icon || p.icon.length < 10) return <div style={{fontSize: '24px'}}>{p.icon || '📦'}</div>;
    return <img src={p.icon} alt="" style={{width: '35px', height: '35px', borderRadius: '5px', objectFit: 'cover'}}/>;
  };

  if (!isLoggedIn) {
    return (
      <div style={loginOverlay}>
        <form onSubmit={handleLogin} style={loginBox}>
          <div style={{ fontSize: '40px', marginBottom: '10px', display: 'flex', justifyContent: 'center', gap: '10px' }}>
            <span>🍋</span><span>🍔</span><span>🍗</span>
          </div>
          <h1 style={{ color: '#2dd4bf', margin: '0 0 20px 0', letterSpacing: '4px', fontSize: '32px', fontFamily: 'serif' }}>TEMPTEA</h1>
          <input type="password" placeholder="••••" value={pinInput} onChange={e => setPinInput(e.target.value)} style={inputStyle} autoFocus />
          <button type="submit" style={actionBtn}>LOGIN</button>
        </form>
      </div>
    );
  }

  return (
    <div style={{ display: 'flex', flexDirection: 'column', height: '100vh', backgroundColor: '#1a1d29', color: 'white' }}>
      
      <style>{`
        @media screen {
          #receipt-print { display: none; }
        }
        @media print {
          body * { visibility: hidden; }
          #receipt-print, #receipt-print * { visibility: visible; }
          #receipt-print { 
            position: absolute; 
            left: 50%;
            transform: translateX(-50%);
            top: 0; 
            width: 300px;
            color: black !important; 
            background: white !important; 
            padding: 10px; 
            font-family: 'Courier New', Courier, monospace;
            font-size: 14px;
          }
          .receipt-line {
            border-top: 1px dashed black;
            margin: 8px 0;
            width: 100%;
          }
          .item-row {
            display: flex;
            justify-content: space-between;
            width: 100%;
            margin-bottom: 4px;
          }
          .total-row {
            display: flex;
            justify-content: space-between;
            width: 100%;
            font-weight: bold;
            margin-top: 5px;
          }
        }
      `}</style>
      
      <div id="receipt-print">
          <div style={{ textAlign: 'center' }}>
            <div>{receiptSettings.logo.length < 10 ? receiptSettings.logo : <img src={receiptSettings.logo} style={{width: '50px'}} />}</div>
            <h2 style={{margin: '5px 0'}}>TEMPTEA</h2>
            <p style={{fontSize: '12px'}}>{new Date().toLocaleString()}</p>
          </div>
          
          <div className="receipt-line"></div>
          
          {cart.map((item, idx) => (
            <div key={idx} className="item-row">
              <span style={{flex: 1}}>{item.name}</span>
              <span style={{width: '60px', textAlign: 'right'}}>
                ₱{item.isDiscounted ? (item.price * 0.8).toFixed(2) : Number(item.price).toFixed(2)}
              </span>
            </div>
          ))}
          
          <div className="receipt-line"></div>
          
          <div className="total-row">
            <span>TOTAL:</span>
            <span>₱{calculateCartTotal().toLocaleString(undefined, {minimumFractionDigits: 2})}</span>
          </div>
          
          <div style={{ textAlign: 'center', marginTop: '20px', fontSize: '12px' }}>
            {receiptSettings.footer}
          </div>
      </div>

      <nav style={navStyle}>
        <h2 style={{ color: '#5eead4', margin: 0 }}>TEMPTEA</h2>
        <div style={{ display: 'flex', gap: '10px' }}>
          <button onClick={() => setCurrentMode('cashier')} style={{...navBtn, background: currentMode === 'cashier' ? '#5eead4' : '#2d3345', color: currentMode === 'cashier' ? '#1a1d29' : 'white'}}>Cashier</button>
          <button onClick={() => setCurrentMode('admin')} style={{...navBtn, background: currentMode === 'admin' ? '#f43f5e' : '#2d3345'}}>Admin</button>
          <button onClick={() => setIsLoggedIn(false)} style={{...navBtn, background: '#f43f5e'}}>Logout</button>
        </div>
      </nav>

      <div style={{ display: 'flex', flex: 1, overflow: 'hidden', padding: '20px' }}>
        <div style={{ flex: 3, overflowY: 'auto', paddingRight: '20px' }}>
          {currentMode === 'cashier' ? (
            <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fill, minmax(110px, 1fr))', gap: '15px' }}>
              {products.filter(p => p.available).map(p => (
                <button 
                  key={p.id} 
                  disabled={p.stock <= 0} 
                  onClick={() => { setSelectedProduct(p); setTempOptions({ main: '100% Sugar', sub: 'None', isDiscounted: false }); }} 
                  style={{...itemCard, opacity: p.stock > 0 ? 1 : 0.5}}
                >
                  {renderIcon(p)}
                  <div style={{fontSize: '12px', marginTop: '5px'}}>{p.name}</div>
                  <div style={{color: '#5eead4'}}>₱{p.price}</div>
                  <div style={{fontSize: '10px', color: '#64748b'}}>Stock: {p.stock}</div>
                </button>
              ))}
            </div>
          ) : (
            <div style={{ display: 'flex', flexDirection: 'column', gap: '20px' }}>
              
              <div style={adminBox}>
                <h4 style={{marginTop: 0}}>Receipt Customization</h4>
                <div style={{display:'flex', gap:'10px', alignItems:'center', marginBottom: '10px'}}>
                  <label style={{fontSize: '12px'}}>Shop Logo:</label>
                  <input type="file" onChange={(e) => handleImageUpload(e, 'receipt')} style={{fontSize: '12px'}}/>
                </div>
                <input placeholder="Receipt Footer Message" value={receiptSettings.footer} onChange={e => setReceiptSettings({...receiptSettings, footer: e.target.value})} style={inputField} />
              </div>

              <div style={{...adminBox, borderLeft: '5px solid #5eead4'}}>
                <h4 style={{margin: '0 0 5px 0', color: '#64748b'}}>Today's Sales Summary</h4>
                <div style={{fontSize: '28px', fontWeight: 'bold', color: '#5eead4'}}>₱{getDailyTotal().toLocaleString()}</div>
              </div>

              <div style={adminBox}>
                <h4 style={{marginTop: 0, fontSize: '18px'}}>Add New Product</h4>
                <div style={{display: 'flex', gap: '12px', marginBottom: '12px'}}>
                  <input placeholder="Product Name" value={newItem.name} onChange={e => setNewItem({...newItem, name: e.target.value})} style={{...inputField, flex: 2}} />
                  <input placeholder="Price (₱)" type="number" value={newItem.price} onChange={e => setNewItem({...newItem, price: e.target.value})} style={{...inputField, flex: 1}} />
                  <input placeholder="Stock" type="number" value={newItem.stock} onChange={e => setNewItem({...newItem, stock: e.target.value})} style={{...inputField, flex: 1}} />
                </div>
                <div style={{marginBottom: '12px'}}>
                  <input placeholder="Add-ons (Pearls, Nata, Cheese...)" value={newItem.options} onChange={e => setNewItem({...newItem, options: e.target.value})} style={inputField} />
                </div>
                <div style={{display: 'flex', gap: '12px', alignItems: 'center', marginBottom: '12px'}}>
                  <select value={newItem.type} onChange={e => setNewItem({...newItem, type: e.target.value})} style={{...inputField, flex: 1, padding: '12px'}}>
                    <option value="Drink">Drink 🥤</option>
                    <option value="Food">Food 🍔</option>
                  </select>
                  <div style={{flex: 1, display: 'flex', alignItems: 'center', gap: '10px', background: '#1a1d29', padding: '8px', borderRadius: '6px', border: '1px solid #3f475e'}}>
                    <input type="file" onChange={handleImageUpload} style={{fontSize: '12px', color: 'white'}} />
                  </div>
                </div>
                <button onClick={() => { 
                  if(!newItem.name || !newItem.price) return alert("Fill all fields");
                  setProducts([...products, {...newItem, id: Date.now(), available: true}]); 
                  setNewItem({name:'', price:'', type:'Drink', icon:'📦', options:'', stock: 0});
                }} style={saveBtnLarge}>SAVE PRODUCT</button>
              </div>

              <div style={adminBox}>
                <h4 style={{marginTop: 0}}>Inventory & Stock Management</h4>
                {products.map(p => (
                  <div key={p.id} style={invRow}>
                    <span style={{display: 'flex', alignItems: 'center', gap: '10px'}}>{renderIcon(p)} {p.name} (Qty: {p.stock})</span>
                    <div style={{display: 'flex', gap: '8px'}}>
                      <button onClick={() => setProducts(products.map(i => i.id === p.id ? {...i, available: !i.available} : i))} style={{...actionBtnTiny, background: p.available ? '#10b981' : '#64748b'}}>{p.available ? 'AVAILABLE' : 'O.O.S'}</button>
                      <button onClick={() => setEditingProduct(p)} style={{...actionBtnTiny, background: '#3b82f6'}}>EDIT STOCK</button>
                      <button onClick={() => setProducts(products.filter(i => i.id !== p.id))} style={{...actionBtnTiny, background: '#f43f5e'}}>DELETE</button>
                    </div>
                  </div>
                ))}
              </div>

              <div style={adminBox}>
                <div style={{display: 'flex', justifyContent: 'space-between', alignItems: 'center'}}>
                  <h4 style={{margin: 0}}>Sales Report & Database</h4>
                  <button onClick={exportToExcel} style={{...navBtn, background: '#10b981', color: 'white', fontSize: '11px'}}>📊 EXCEL DOWNLOAD</button>
                </div>
                <div style={{marginTop: '15px', maxHeight: '250px', overflowY: 'auto'}}>
                  <table style={{width: '100%', borderCollapse: 'collapse', fontSize: '12px'}}>
                    <thead><tr style={{color: '#64748b', borderBottom: '1px solid #2d3345'}}><th style={thS}>Date</th><th style={thS}>Items</th><th style={thS}>Total</th></tr></thead>
                    <tbody>
                      {sales.map(s => (
                        <tr key={s.id} style={{borderBottom: '1px solid #2d3345'}}>
                          <td style={tdS}>{s.date}</td>
                          <td style={tdS}>{s.items}</td>
                          <td style={{...tdS, color: '#5eead4', fontWeight: 'bold'}}>₱{s.total.toLocaleString()}</td>
                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>
              </div>
            </div>
          )}
        </div>

        <div style={cartSidebar}>
          <h4 style={{marginTop: 0, borderBottom: '1px solid #2d3345', paddingBottom: '10px'}}>Current Cart</h4>
          <div style={{flex: 1, overflowY: 'auto'}}>
            {cart.map(item => (
              <div key={item.cartId} style={cartItemStyle}>
                <div style={{display: 'flex', justifyContent: 'space-between'}}>
                  <b style={{fontSize: '13px'}}>{item.name} {item.isDiscounted && <span style={{color: '#f43f5e'}}>(-20%)</span>}</b> 
                  <span>₱{item.isDiscounted ? (item.price * 0.8).toFixed(2) : Number(item.price).toFixed(2)}</span>
                </div>
                <div style={{fontSize: '10px', color: '#64748b'}}>{item.details}</div>
              </div>
            ))}
          </div>
          <div style={{borderTop: '2px solid #2d3345', paddingTop: '15px'}}>
            <div style={{display: 'flex', justifyContent: 'space-between', marginBottom: '10px'}}>
              <span>Grand Total:</span>
              <span style={{color: '#5eead4', fontWeight: 'bold', fontSize: '20px'}}>₱{calculateCartTotal().toLocaleString()}</span>
            </div>
            <button onClick={handlePrintAndComplete} style={printBtn}>PRINT & COMPLETE</button>
            <button onClick={() => setCart([])} style={{background: 'transparent', border: 'none', color: '#64748b', width: '100%', marginTop: '10px', cursor: 'pointer', fontSize: '12px'}}>Clear Cart</button>
          </div>
        </div>
      </div>

      {editingProduct && (
        <div style={modalOverlay}>
          <div style={modalContent}>
            <h3>Edit Product & Stock</h3>
            <label style={labelS}>Name</label>
            <input value={editingProduct.name} onChange={e => setEditingProduct({...editingProduct, name: e.target.value})} style={inputField}/>
            <label style={labelS}>Price</label>
            <input type="number" value={editingProduct.price} onChange={e => setEditingProduct({...editingProduct, price: e.target.value})} style={inputField}/>
            <label style={labelS}>Current Stock</label>
            <input type="number" value={editingProduct.stock} onChange={e => setEditingProduct({...editingProduct, stock: e.target.value})} style={inputField}/>
            <label style={labelS}>Add-ons</label>
            <input value={editingProduct.options} onChange={e => setEditingProduct({...editingProduct, options: e.target.value})} style={inputField}/>
            <button onClick={() => { setProducts(products.map(p => p.id === editingProduct.id ? editingProduct : p)); setEditingProduct(null); }} style={saveBtnLarge}>SAVE CHANGES</button>
            <button onClick={() => setEditingProduct(null)} style={{...saveBtnLarge, background: '#f43f5e', marginTop: '8px'}}>CANCEL</button>
          </div>
        </div>
      )}

      {selectedProduct && (
        <div style={modalOverlay}>
          <div style={modalContent}>
            <h3>{selectedProduct.name}</h3>
            <p style={{fontSize: '12px', color: '#5eead4'}}>In Stock: {selectedProduct.stock}</p>
            {selectedProduct.type === 'Drink' && (
              <select style={{...inputField, marginBottom: '10px'}} onChange={e => setTempOptions({...tempOptions, main: e.target.value})}>
                <option>100% Sugar</option><option>50% Sugar</option><option>0% Sugar</option>
              </select>
            )}
            <select style={inputField} onChange={e => setTempOptions({...tempOptions, sub: e.target.value})}>
              <option value="None">No Add-ons</option>
              {selectedProduct.options?.split(',').map((opt, i) => <option key={i} value={opt.trim()}>{opt.trim()}</option>)}
            </select>
            
            <div style={{display: 'flex', alignItems: 'center', gap: '10px', margin: '15px 0', padding: '10px', background: '#0f172a', borderRadius: '8px', border: '1px solid #334155'}}>
              <input type="checkbox" id="disc" checked={tempOptions.isDiscounted} onChange={e => setTempOptions({...tempOptions, isDiscounted: e.target.checked})} style={{width: '18px', height: '18px'}}/>
              <label htmlFor="disc" style={{fontSize: '14px', cursor: 'pointer'}}>Apply 20% Discount</label>
            </div>

            <button onClick={() => {
              const details = selectedProduct.type === 'Drink' ? `${tempOptions.main}, Add-on: ${tempOptions.sub}` : `Add-on: ${tempOptions.sub}`;
              setCart([...cart, { ...selectedProduct, details, isDiscounted: tempOptions.isDiscounted, cartId: Date.now() }]);
              setSelectedProduct(null);
            }} style={saveBtnLarge}>ADD TO CART</button>
            <button onClick={() => setSelectedProduct(null)} style={{...saveBtnLarge, background: '#f43f5e', marginTop: '8px'}}>CANCEL</button>
          </div>
        </div>
      )}
    </div>
  );
}

const navStyle = { padding: '20px', background: '#1a1d29', display: 'flex', justifyContent: 'space-between', alignItems: 'center' };
const navBtn = { padding: '8px 16px', borderRadius: '6px', border: 'none', cursor: 'pointer', fontWeight: 'bold' };
const itemCard = { background: '#2d3345', padding: '15px', borderRadius: '10px', border: '1px solid #3f475e', color: 'white', cursor: 'pointer', textAlign: 'center' };
const adminBox = { background: '#242938', padding: '20px', borderRadius: '10px', border: '1px solid #2d3345' };
const inputField = { width: '100%', padding: '10px', background: '#1a1d29', color: 'white', border: '1px solid #3f475e', borderRadius: '6px', boxSizing: 'border-box' };
const saveBtnLarge = { width: '100%', padding: '15px', background: '#5eead4', color: '#1a1d29', border: 'none', borderRadius: '8px', fontWeight: 'bold', cursor: 'pointer', fontSize: '16px' };
const invRow = { display: 'flex', justifyContent: 'space-between', alignItems: 'center', padding: '10px 0', borderBottom: '1px solid #2d3345' };
const actionBtnTiny = { padding: '6px 12px', border: 'none', borderRadius: '4px', color: 'white', fontSize: '10px', fontWeight: 'bold', cursor: 'pointer' };
const cartSidebar = { width: '320px', background: '#242938', padding: '20px', borderRadius: '10px', display: 'flex', flexDirection: 'column' };
const cartItemStyle = { background: '#1a1d29', padding: '12px', borderRadius: '8px', marginBottom: '8px' };
const printBtn = { width: '100%', padding: '15px', background: '#448c7e', color: 'white', border: 'none', borderRadius: '8px', cursor: 'pointer', fontWeight: 'bold', fontSize: '14px' };
const modalOverlay = { position: 'fixed', top: 0, left: 0, right: 0, bottom: 0, background: 'rgba(0,0,0,0.85)', display: 'flex', alignItems: 'center', justifyContent: 'center', zIndex: 1000 };
const modalContent = { background: '#1e293b', padding: '25px', borderRadius: '15px', width: '340px' };
const loginOverlay = { height: '100vh', background: '#0f172a', display: 'flex', justifyContent: 'center', alignItems: 'center' };
const loginBox = { background: '#1e293b', padding: '50px', borderRadius: '24px', textAlign: 'center', boxShadow: '0 20px 25px -5px rgba(0, 0, 0, 0.5)' };
const inputStyle = { padding: '14px', background: '#0f172a', color: 'white', border: '1px solid #334155', borderRadius: '10px', display: 'block', margin: '0 auto 20px auto', width: '200px', textAlign: 'center', fontSize: '18px', letterSpacing: '8px' };
const actionBtn = { padding: '12px 40px', background: '#2dd4bf', border: 'none', borderRadius: '10px', fontWeight: 'bold', cursor: 'pointer', fontSize: '14px', color: '#0f172a' };
const thS = { textAlign: 'left', padding: '10px' };
const tdS = { padding: '10px' };
const labelS = { fontSize: '11px', color: '#64748b', display: 'block', margin: '10px 0 5px 0' };

export default App;
