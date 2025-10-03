# .NET website feedback

This GitHub repository is for issues encountered on <https://dotnet.microsoft.com>.
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Fullscreen HTML File Manager with Editable Titles</title>
<style>
body { font-family: 'Segoe UI', sans-serif; margin: 0; padding: 20px; background: #f2f2f2; }
.container { max-width: 900px; margin: auto; background: #fff; padding: 20px; border-radius: 10px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
h1 { text-align: center; margin-bottom: 20px; color: #333; }
label { display: block; margin-top: 10px; font-weight: bold; }
input[type="file"], input[type="text"], textarea { width: 100%; padding: 8px; margin-top: 5px; border-radius: 5px; border: 1px solid #ccc; }
button { padding: 8px 15px; margin-top: 10px; border: none; border-radius: 5px; background: #4CAF50; color: #fff; font-size: 16px; cursor: pointer; transition: 0.2s; }
button:hover { background: #45a049; }
table { width: 100%; border-collapse: collapse; margin-top: 20px; }
th, td { border: 1px solid #ccc; padding: 10px; text-align: left; }
th { background-color: #f4f4f4; cursor: pointer; }
tr:nth-child(even) { background-color: #f9f9f9; }

/* Fullscreen Viewer */
#fullscreenViewer { display: none; position: fixed; top:0; left:0; width: 100%; height: 100%; background: white; z-index: 9999; }
#fullscreenViewer iframe { width: 100%; height: 100%; border: none; }
#backBtn {
    position: fixed;
    top: 20px; 
    right: 20px;
    padding: 5px 10px;
    background-color: #800080; /* Purple */
    color: white;
    border: none;
    border-radius: 4px;
    font-size: 14px;
    cursor: pointer;
    z-index: 10000;
    opacity: 0.9;
    display: inline-block;
    width: auto;
    height: auto;
}
#backBtn:hover { opacity: 1; }

@media (max-width: 600px){ button { width: 100%; margin-top: 5px; } }

#titlePage { max-width: 600px; margin: 50px auto; background: #fff; padding: 30px; border-radius: 10px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
#titleList { margin-top: 20px; text-align: left; }
#titleList button { margin-top: 5px; margin-right:5px; background-color:#2196F3; color:white; border:none; border-radius:4px; padding:5px 10px; cursor:pointer; position: relative; }
#titleList button:hover { opacity:0.9; }

/* Context Menu for Titles */
#titleContextMenu {
  display: none;
  position: absolute;
  background: #fff;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.2);
  z-index: 9999;
}
#titleContextMenu button {
  display: block;
  width: 100%;
  padding: 5px 10px;
  border: none;
  background: none;
  text-align: left;
  cursor: pointer;
  color: black; /* ফন্ট কালার কালো করা হলো */
}
#titleContextMenu button:hover { background: #f2f2f2; }
</style>
</head>
<body>

<!-- Step 1: Title Page -->
<div id="titlePage">
  <h1>Manage Titles</h1>
  <label>Create New Title:</label>
  <input type="text" id="newTitleInput" placeholder="Type new title">
  <button id="addTitleBtn">Add Title</button>
  <div id="titleList">
    <h3>Existing Titles:</h3>
    <div id="titlesContainer"></div>
  </div>
</div>

<!-- Step 2: Manager UI -->
<div class="container" id="managerUI" style="display:none;">
<h1 id="managerHeader">Fullscreen HTML File Manager</h1>

<label>Select HTML file:</label>
<input type="file" id="fileInput" accept=".html">

<label>Enter file name:</label>
<input type="text" id="fileName" placeholder="Enter name">

<button id="uploadBtn">Upload HTML File</button>

<h2>Uploaded Files</h2>
<table id="fileTable">
  <thead>
    <tr>
      <th>#</th>
      <th>Name</th>
      <th>Actions</th>
    </tr>
  </thead>
  <tbody></tbody>
</table>
<button id="backToTitlesBtn" style="background:#2196F3;">Back to Titles</button>
</div>

<!-- Fullscreen Viewer -->
<div id="fullscreenViewer">
  <button id="backBtn">Back</button>
  <iframe id="viewer"></iframe>
</div>

<!-- Context Menu -->
<div id="titleContextMenu">
  <button id="renameTitleBtn">Rename</button>
  <button id="deleteTitleBtn">Delete</button>
</div>

<script>
// UI Elements
const titlePage = document.getElementById('titlePage');
const newTitleInput = document.getElementById('newTitleInput');
const addTitleBtn = document.getElementById('addTitleBtn');
const titlesContainer = document.getElementById('titlesContainer');
const titleContextMenu = document.getElementById('titleContextMenu');
let selectedTitleButton = null;

const managerUI = document.getElementById('managerUI');
const managerHeader = document.getElementById('managerHeader');
const fileInput = document.getElementById('fileInput');
const fileNameInput = document.getElementById('fileName');
const uploadBtn = document.getElementById('uploadBtn');
const fileTableBody = document.querySelector('#fileTable tbody');
const backToTitlesBtn = document.getElementById('backToTitlesBtn');
const fullscreenViewer = document.getElementById('fullscreenViewer');
const viewer = document.getElementById('viewer');
const backBtn = document.getElementById('backBtn');

let db;
const dbName = "HTMLManagerDB";
let currentTitle = "";

// IndexedDB Setup
function openDB() {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open(dbName, 1);
    request.onerror = ()=>reject("DB open failed");
    request.onsuccess = (e)=>{ db = e.target.result; resolve(); };
    request.onupgradeneeded = (e)=>{
      db = e.target.result;
      if(!db.objectStoreNames.contains("titles")){
        db.createObjectStore("titles", { keyPath: "title" });
      }
    };
  });
}

// Add new title
addTitleBtn.onclick = async ()=>{
  const title = newTitleInput.value.trim();
  if(title === "") return alert("Title cannot be empty");
  const tx = db.transaction(["titles"], "readwrite");
  const store = tx.objectStore("titles");
  store.put({ title: title, files: [] });
  newTitleInput.value = '';
  await loadTitles();
};

// Load all titles
async function loadTitles(){
  titlesContainer.innerHTML = '';
  const tx = db.transaction(["titles"], "readonly");
  const store = tx.objectStore("titles");
  const req = store.getAll();
  req.onsuccess = ()=>{
    req.result.forEach(item=>{
      const btn = document.createElement('button');
      btn.textContent = item.title;
      btn.onclick = ()=>enterTitle(item.title);

      // Right-click (context menu) for rename/delete
      btn.oncontextmenu = (e)=>{
        e.preventDefault();
        selectedTitleButton = btn;
        titleContextMenu.style.display = 'block';
        titleContextMenu.style.left = e.pageX + 'px';
        titleContextMenu.style.top = e.pageY + 'px';
      };

      titlesContainer.appendChild(btn);
    });
  };
}

// Context menu actions
document.getElementById('renameTitleBtn').onclick = async ()=>{
  if(!selectedTitleButton) return;
  const oldTitle = selectedTitleButton.textContent;
  const newTitle = prompt("Enter new title name:", oldTitle);
  if(newTitle && newTitle.trim()!==""){
    const tx = db.transaction(["titles"], "readwrite");
    const store = tx.objectStore("titles");
    const req = store.get(oldTitle);
    req.onsuccess = ()=>{
      const data = req.result;
      store.delete(oldTitle);
      data.title = newTitle.trim();
      store.put(data);
      loadTitles();
    };
  }
  titleContextMenu.style.display = 'none';
};

document.getElementById('deleteTitleBtn').onclick = async ()=>{
  if(!selectedTitleButton) return;
  if(confirm('Delete this title and all its files?')){
    const title = selectedTitleButton.textContent;
    const tx = db.transaction(["titles"], "readwrite");
    const store = tx.objectStore("titles");
    store.delete(title);
    loadTitles();
  }
  titleContextMenu.style.display = 'none';
};

// Hide context menu on click outside
document.addEventListener('click', ()=>{ titleContextMenu.style.display = 'none'; });

// Enter a title
function enterTitle(title){
  currentTitle = title;
  managerHeader.textContent = `Title: ${title}`;
  titlePage.style.display = 'none';
  managerUI.style.display = 'block';
  updateTable();
}

// Get files for current title
function getFiles(){
  return new Promise((resolve,reject)=>{
    const tx = db.transaction(["titles"], "readonly");
    const store = tx.objectStore("titles");
    const req = store.get(currentTitle);
    req.onsuccess = ()=>{
      resolve(req.result ? req.result.files : []);
    };
    req.onerror = ()=>reject("Failed to get files");
  });
}

// Save files for current title
function saveFiles(files){
  return new Promise((resolve,reject)=>{
    const tx = db.transaction(["titles"], "readwrite");
    const store = tx.objectStore("titles");
    const reqGet = store.get(currentTitle);
    reqGet.onsuccess = ()=>{
      const data = reqGet.result;
      data.files = files;
      const reqPut = store.put(data);
      reqPut.onsuccess = ()=>resolve();
      reqPut.onerror = ()=>reject("Failed to save files");
    };
    reqGet.onerror = ()=>reject("Failed to get files for saving");
  });
}

// Update table
async function updateTable(){
  fileTableBody.innerHTML = '';
  const files = await getFiles();
  files.forEach((file, idx)=>{
    const tr = document.createElement('tr');

    const tdIndex = document.createElement('td'); tdIndex.textContent = idx+1;
    const tdName = document.createElement('td'); tdName.textContent = file.name;

    const tdActions = document.createElement('td');
    const openBtn = document.createElement('button');
    openBtn.textContent = 'Open';
    openBtn.onclick = ()=>{
      fullscreenViewer.style.display = 'block';
      managerUI.style.display = 'none';
      const iframeDoc = viewer.contentDocument || viewer.contentWindow.document;
      iframeDoc.open();
      iframeDoc.write(file.content);
      iframeDoc.close();
    };

    const renameBtn = document.createElement('button');
    renameBtn.textContent = 'Rename';
    renameBtn.style.backgroundColor = '#2196F3';
    renameBtn.onclick = async ()=>{
      const newName = prompt("Enter new file name:", file.name);
      if(newName && newName.trim() !== ""){
        const allFiles = await getFiles();
        allFiles[idx].name = newName.trim();
        await saveFiles(allFiles);
        updateTable();
      }
    };

    const deleteBtn = document.createElement('button');
    deleteBtn.textContent = 'Delete';
    deleteBtn.style.backgroundColor = '#f44336';
    deleteBtn.onclick = async ()=>{
      if(confirm('Delete this file?')){
        const allFiles = await getFiles();
        allFiles.splice(idx,1);
        await saveFiles(allFiles);
        updateTable();
      }
    };

    tdActions.appendChild(openBtn);
    tdActions.appendChild(renameBtn);
    tdActions.appendChild(deleteBtn);

    tr.appendChild(tdIndex);
    tr.appendChild(tdName);
    tr.appendChild(tdActions);
    fileTableBody.appendChild(tr);
  });
}

// Upload file
uploadBtn.onclick = async ()=>{
  const file = fileInput.files[0];
  if(!file) return alert("Select a file");
  const name = fileNameInput.value.trim() || file.name;
  const reader = new FileReader();
  reader.onload = async e=>{
    const allFiles = await getFiles();
    allFiles.push({name:name, content:e.target.result, type:file.type});
    await saveFiles(allFiles);
    updateTable();
    fileInput.value = '';
    fileNameInput.value = '';
  };
  reader.readAsText(file);
};

// Fullscreen back button
backBtn.onclick = ()=>{
  fullscreenViewer.style.display = 'none';
  managerUI.style.display = 'block';
  viewer.src = 'about:blank';
};

// Back to titles
backToTitlesBtn.onclick = ()=>{
  managerUI.style.display = 'none';
  titlePage.style.display = 'block';
  currentTitle = '';
  loadTitles();
};

// Initialize
(async ()=>{
  await openDB();
  loadTitles();
})();
</script>
</body>
</html>
