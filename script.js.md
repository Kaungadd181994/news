const feed = document.getElementById('feed');
const catBar = document.getElementById('category-bar');

 
const categories = {
  'Business': ['post1.md','post2.md'],
  'Tech': ['another-tech-post.md'],
  'General': ['general-news.md']
}; 
 
function createCard(category, filename) {
  const title = filename.replace('.md','').replace(/-/g,' ');
  const card = document.createElement('div');
  card.className = 'card';
  card.textContent = title;
  card.onclick = () => showArticle(category, filename, title);
  card.dataset.category = category;
  feed.appendChild(card);
}

function showArticle(category, filename, title) {
  fetch(`${category.toLowerCase()}/${filename}`)
    .then(res => res.text())
    .then(md => {
      feed.innerHTML = '';
      catBar.style.display = 'none';
      const html = marked.parse(md);
      feed.insertAdjacentHTML('afterbegin', `
        <button class="back-btn" onclick="location.reload()">← Back</button>
        <article class="content"><h1>${title}</h1>${html}</article>
      `);
      document.querySelector('.back-btn').style.display = 'block';
    });
}

function filterBy(cat) {
  document.querySelectorAll('.card').forEach(card => {
    card.style.display = (cat === 'All' || card.dataset.category === cat) ? '' : 'none';
  });
}

function init() {
  ['All', ...Object.keys(categories)].forEach(cat => {
    const btn = document.createElement('button');
    btn.textContent = cat;
    btn.onclick = () => filterBy(cat);
    catBar.appendChild(btn);
  });

  Object.entries(categories).forEach(([cat, files]) => {
    files.forEach(fn => createCard(cat, fn));
  });
}

window.onload = init;
<!--stackedit_data:
eyJoaXN0b3J5IjpbODY3MjM2ODM4XX0=
-->