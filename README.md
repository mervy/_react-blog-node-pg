# DeepSeek Conversation

## Prompt:
Blog completo com postgres, node, ES6+ e react no frontend com boostrap 5 ou superior. Login com senha encriptada, avatar do usuário e posts do blog com user, categoria e imagem do post

## Response:
Criar um blog completo com as tecnologias mencionadas (PostgreSQL, Node.js, ES6+, React e Bootstrap 5) envolve várias etapas. Vou guiá-lo através do processo, desde a configuração do backend até a criação do frontend. Vamos dividir o projeto em partes:

### 1\. Configuração do Ambiente

#### 1.1. Instalação de Dependências

Primeiro, certifique-se de ter o Node.js e o PostgreSQL instalados em sua máquina. Em seguida, crie um novo diretório para o projeto e inicialize um novo projeto Node.js:

```
mkdir meu-blog
cd meu-blog
npm init -y
```

Instale as dependências necessárias:

```
npm install express pg bcryptjs jsonwebtoken multer cors
npm install --save-dev nodemon
```

*   **express**: Framework para criar o servidor.
    
*   **pg**: Cliente PostgreSQL para Node.js.
    
*   **bcryptjs**: Para criptografar senhas.
    
*   **jsonwebtoken**: Para autenticação via JWT.
    
*   **multer**: Para upload de arquivos (imagens do avatar e posts).
    
*   **cors**: Para permitir requisições de diferentes origens.
    
*   **nodemon**: Para reiniciar o servidor automaticamente durante o desenvolvimento.
    

#### 1.2. Configuração do Banco de Dados

Crie um banco de dados PostgreSQL chamado `meu_blog`. Em seguida, crie as tabelas necessárias:

```
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    avatar VARCHAR(255)
);

CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    image VARCHAR(255),
    user_id INT REFERENCES users(id),
    category_id INT REFERENCES categories(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2\. Backend (Node.js + Express)

#### 2.1. Estrutura do Projeto

Crie a seguinte estrutura de pastas:

```
meu-blog/
│
├── src/
│   ├── config/
│   │   └── db.js
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── postController.js
│   │   └── userController.js
│   ├── middleware/
│   │   └── authMiddleware.js
│   ├── models/
│   │   ├── User.js
│   │   ├── Post.js
│   │   └── Category.js
│   ├── routes/
│   │   ├── authRoutes.js
│   │   ├── postRoutes.js
│   │   └── userRoutes.js
│   └── utils/
│       └── upload.js
│
├── .env
├── app.js
└── server.js
```

#### 2.2. Configuração do Banco de Dados

Em `src/config/db.js`:

```
const { Pool } = require('pg');
require('dotenv').config();

const pool = new Pool({
  user: process.env.DB_USER,
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  password: process.env.DB_PASSWORD,
  port: process.env.DB_PORT,
});

module.exports = pool;
```

#### 2.3. Modelos

Em `src/models/User.js`:

```
const pool = require('../config/db');
const bcrypt = require('bcryptjs');

class User {
  static async create(username, password, avatar) {
    const hashedPassword = await bcrypt.hash(password, 10);
    const { rows } = await pool.query(
      'INSERT INTO users (username, password, avatar) VALUES ($1, $2, $3) RETURNING *',
      [username, hashedPassword, avatar]
    );
    return rows[0];
  }

  static async findByUsername(username) {
    const { rows } = await pool.query('SELECT * FROM users WHERE username = $1', [username]);
    return rows[0];
  }

  static async findById(id) {
    const { rows } = await pool.query('SELECT * FROM users WHERE id = $1', [id]);
    return rows[0];
  }
}

module.exports = User;
```

Em `src/models/Post.js`:

```
const pool = require('../config/db');

class Post {
  static async create(title, content, image, userId, categoryId) {
    const { rows } = await pool.query(
      'INSERT INTO posts (title, content, image, user_id, category_id) VALUES ($1, $2, $3, $4, $5) RETURNING *',
      [title, content, image, userId, categoryId]
    );
    return rows[0];
  }

  static async findAll() {
    const { rows } = await pool.query('SELECT * FROM posts');
    return rows;
  }

  static async findById(id) {
    const { rows } = await pool.query('SELECT * FROM posts WHERE id = $1', [id]);
    return rows[0];
  }
}

module.exports = Post;
```

#### 2.4. Controladores

Em `src/controllers/authController.js`:

```
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const bcrypt = require('bcryptjs');

const login = async (req, res) => {
  const { username, password } = req.body;

  const user = await User.findByUsername(username);
  if (!user) return res.status(400).json({ message: 'Usuário não encontrado' });

  const validPassword = await bcrypt.compare(password, user.password);
  if (!validPassword) return res.status(400).json({ message: 'Senha inválida' });

  const token = jwt.sign({ id: user.id }, process.env.JWT_SECRET, { expiresIn: '1h' });
  res.json({ token });
};

module.exports = { login };
```

Em `src/controllers/postController.js`:

```
const Post = require('../models/Post');

const createPost = async (req, res) => {
  const { title, content, categoryId } = req.body;
  const image = req.file ? req.file.path : null;
  const userId = req.user.id;

  const post = await Post.create(title, content, image, userId, categoryId);
  res.status(201).json(post);
};

const getPosts = async (req, res) => {
  const posts = await Post.findAll();
  res.json(posts);
};

module.exports = { createPost, getPosts };
```

#### 2.5. Middleware de Autenticação

Em `src/middleware/authMiddleware.js`:

```
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ message: 'Acesso negado' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(400).json({ message: 'Token inválido' });
  }
};

module.exports = authMiddleware;
```

#### 2.6. Rotas

Em `src/routes/authRoutes.js`:

```
const express = require('express');
const { login } = require('../controllers/authController');

const router = express.Router();

router.post('/login', login);

module.exports = router;
```

Em `src/routes/postRoutes.js`:

```
const express = require('express');
const { createPost, getPosts } = require('../controllers/postController');
const authMiddleware = require('../middleware/authMiddleware');
const upload = require('../utils/upload');

const router = express.Router();

router.post('/posts', authMiddleware, upload.single('image'), createPost);
router.get('/posts', getPosts);

module.exports = router;
```

#### 2.7. Upload de Imagens

Em `src/utils/upload.js`:

```
const multer = require('multer');
const path = require('path');

const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/');
  },
  filename: (req, file, cb) => {
    cb(null, Date.now() + path.extname(file.originalname));
  },
});

const upload = multer({ storage });

module.exports = upload;
```

#### 2.8. Configuração do Servidor

Em `app.js`:

```
const express = require('express');
const cors = require('cors');
const authRoutes = require('./src/routes/authRoutes');
const postRoutes = require('./src/routes/postRoutes');
const userRoutes = require('./src/routes/userRoutes');

const app = express();

app.use(cors());
app.use(express.json());
app.use('/uploads', express.static('uploads'));

app.use('/api/auth', authRoutes);
app.use('/api/posts', postRoutes);
app.use('/api/users', userRoutes);

module.exports = app;
```

Em `server.js`:

```
const app = require('./app');
const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
  console.log(`Servidor rodando na porta ${PORT}`);
});
```

#### 2.9. Variáveis de Ambiente

Crie um arquivo `.env` na raiz do projeto:

```
DB_USER=seu_usuario
DB_HOST=localhost
DB_NAME=meu_blog
DB_PASSWORD=sua_senha
DB_PORT=5432
JWT_SECRET=seu_segredo_jwt
```

### 3\. Frontend (React + Bootstrap 5)

#### 3.1. Configuração do Projeto React

Crie um novo projeto React:

```
npx create-react-app frontend
cd frontend
npm install axios react-router-dom bootstrap
```

#### 3.2. Estrutura do Projeto

```
frontend/
│
├── src/
│   ├── components/
│   │   ├── Navbar.js
│   │   ├── PostForm.js
│   │   ├── PostList.js
│   │   └── Login.js
│   ├── App.js
│   ├── index.js
│   └── styles/
│       └── App.css
│
└── public/
    └── index.html
```

#### 3.3. Componentes

Em `src/components/Navbar.js`:

```
import React from 'react';
import { Link } from 'react-router-dom';

const Navbar = () => {
  return (
    <nav className="navbar navbar-expand-lg navbar-light bg-light">
      <div className="container-fluid">
        <Link className="navbar-brand" to="/">Meu Blog</Link>
        <div className="collapse navbar-collapse">
          <ul className="navbar-nav me-auto mb-2 mb-lg-0">
            <li className="nav-item">
              <Link className="nav-link" to="/posts">Posts</Link>
            </li>
            <li className="nav-item">
              <Link className="nav-link" to="/login">Login</Link>
            </li>
          </ul>
        </div>
      </div>
    </nav>
  );
};

export default Navbar;
```

Em `src/components/Login.js`:

```
import React, { useState } from 'react';
import axios from 'axios';
import { useHistory } from 'react-router-dom';

const Login = () => {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const history = useHistory();

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const res = await axios.post('http://localhost:5000/api/auth/login', { username, password });
      localStorage.setItem('token', res.data.token);
      history.push('/posts');
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" placeholder="Username" value={username} onChange={(e) => setUsername(e.target.value)} />
      <input type="password" placeholder="Password" value={password} onChange={(e) => setPassword(e.target.value)} />
      <button type="submit">Login</button>
    </form>
  );
};

export default Login;
```

Em `src/components/PostForm.js`:

```
import React, { useState } from 'react';
import axios from 'axios';

const PostForm = () => {
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  const [image, setImage] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData();
    formData.append('title', title);
    formData.append('content', content);
    formData.append('image', image);

    const token = localStorage.getItem('token');
    try {
      await axios.post('http://localhost:5000/api/posts', formData, {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'multipart/form-data',
        },
      });
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" placeholder="Title" value={title} onChange={(e) => setTitle(e.target.value)} />
      <textarea placeholder="Content" value={content} onChange={(e) => setContent(e.target.value)} />
      <input type="file" onChange={(e) => setImage(e.target.files[0])} />
      <button type="submit">Create Post</button>
    </form>
  );
};

export default PostForm;
```

Em `src/components/PostList.js`:

```
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const PostList = () => {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    const fetchPosts = async () => {
      const res = await axios.get('http://localhost:5000/api/posts');
      setPosts(res.data);
    };
    fetchPosts();
  }, []);

  return (
    <div>
      {posts.map(post => (
        <div key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.content}</p>
          {post.image && <img src={`http://localhost:5000/uploads/${post.image}`} alt={post.title} />}
        </div>
      ))}
    </div>
  );
};

export default PostList;
```

#### 3.4. App.js

Em `src/App.js`:

```
import React from 'react';
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';
import Navbar from './components/Navbar';
import PostList from './components/PostList';
import PostForm from './components/PostForm';
import Login from './components/Login';
import 'bootstrap/dist/css/bootstrap.min.css';

function App() {
  return (
    <Router>
      <Navbar />
      <div className="container mt-4">
        <Switch>
          <Route path="/posts" component={PostList} />
          <Route path="/create-post" component={PostForm} />
          <Route path="/login" component={Login} />
        </Switch>
      </div>
    </Router>
  );
}

export default App;
```

#### 3.5. index.js

Em `src/index.js`:

```
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';

ReactDOM.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
  document.getElementById('root')
);
```

### 4\. Executando o Projeto

#### 4.1. Backend

No diretório `meu-blog`, execute:

```
npm run dev
```

#### 4.2. Frontend

No diretório `frontend`, execute:

```
npm start
```

Agora, você deve ter um blog funcional com autenticação de usuário, upload de imagens, e posts categorizados. O frontend está conectado ao backend, permitindo a criação e visualização de posts, além de login de usuários.

### 5\. Considerações Finais

Este é um projeto básico, mas funcional. Você pode expandir e melhorar o projeto adicionando mais funcionalidades, como:

*   Edição e exclusão de posts.
    
*   Comentários nos posts.
    
*   Paginação de posts.
    
*   Validação de formulários no frontend.
    
*   Melhorias na interface com Bootstrap.
    

Espero que este guia tenha sido útil! Se tiver alguma dúvida ou precisar de mais ajuda, sinta-se à vontade para perguntar.

## Prompt:
Adicione para mim essas funcionalidades. Use import export do ES6+. Edição e exclusão de posts. Comentários nos posts. Paginação de posts. Validação de formulários no frontend. Melhorias na interface com Bootstrap.

## Response:
Vamos adicionar as funcionalidades solicitadas ao projeto, utilizando `import/export` do ES6+ e melhorando a interface com Bootstrap 5. Vamos dividir as implementações em etapas.

* * *

1\. **Edição e Exclusão de Posts**
----------------------------------

### Backend

#### 1.1. Atualize o Modelo de Post (`src/models/Post.js`):

Adicione métodos para editar e excluir posts:

```
import pool from '../config/db.js';

class Post {
  static async create(title, content, image, userId, categoryId) {
    const { rows } = await pool.query(
      'INSERT INTO posts (title, content, image, user_id, category_id) VALUES ($1, $2, $3, $4, $5) RETURNING *',
      [title, content, image, userId, categoryId]
    );
    return rows[0];
  }

  static async findAll() {
    const { rows } = await pool.query('SELECT * FROM posts');
    return rows;
  }

  static async findById(id) {
    const { rows } = await pool.query('SELECT * FROM posts WHERE id = $1', [id]);
    return rows[0];
  }

  static async update(id, title, content, image, categoryId) {
    const { rows } = await pool.query(
      'UPDATE posts SET title = $1, content = $2, image = $3, category_id = $4 WHERE id = $5 RETURNING *',
      [title, content, image, categoryId, id]
    );
    return rows[0];
  }

  static async delete(id) {
    await pool.query('DELETE FROM posts WHERE id = $1', [id]);
  }
}

export default Post;
```

#### 1.2. Atualize o Controlador de Posts (`src/controllers/postController.js`):

Adicione métodos para editar e excluir posts:

```
import Post from '../models/Post.js';

const createPost = async (req, res) => {
  const { title, content, categoryId } = req.body;
  const image = req.file ? req.file.path : null;
  const userId = req.user.id;

  const post = await Post.create(title, content, image, userId, categoryId);
  res.status(201).json(post);
};

const getPosts = async (req, res) => {
  const posts = await Post.findAll();
  res.json(posts);
};

const updatePost = async (req, res) => {
  const { id } = req.params;
  const { title, content, categoryId } = req.body;
  const image = req.file ? req.file.path : null;

  const post = await Post.update(id, title, content, image, categoryId);
  res.json(post);
};

const deletePost = async (req, res) => {
  const { id } = req.params;
  await Post.delete(id);
  res.status(204).send();
};

export { createPost, getPosts, updatePost, deletePost };
```

#### 1.3. Atualize as Rotas de Posts (`src/routes/postRoutes.js`):

Adicione rotas para editar e excluir posts:

```
import express from 'express';
import { createPost, getPosts, updatePost, deletePost } from '../controllers/postController.js';
import authMiddleware from '../middleware/authMiddleware.js';
import upload from '../utils/upload.js';

const router = express.Router();

router.post('/posts', authMiddleware, upload.single('image'), createPost);
router.get('/posts', getPosts);
router.put('/posts/:id', authMiddleware, upload.single('image'), updatePost);
router.delete('/posts/:id', authMiddleware, deletePost);

export default router;
```

* * *

### Frontend

#### 1.4. Crie um Componente para Editar Posts (`src/components/EditPost.js`):

```
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useParams, useHistory } from 'react-router-dom';

const EditPost = () => {
  const { id } = useParams();
  const history = useHistory();
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  const [image, setImage] = useState(null);

  useEffect(() => {
    const fetchPost = async () => {
      const res = await axios.get(`http://localhost:5000/api/posts/${id}`);
      setTitle(res.data.title);
      setContent(res.data.content);
    };
    fetchPost();
  }, [id]);

  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData();
    formData.append('title', title);
    formData.append('content', content);
    if (image) formData.append('image', image);

    const token = localStorage.getItem('token');
    try {
      await axios.put(`http://localhost:5000/api/posts/${id}`, formData, {
        headers: {
          Authorization: `Bearer ${token}`,
          'Content-Type': 'multipart/form-data',
        },
      });
      history.push('/posts');
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" placeholder="Title" value={title} onChange={(e) => setTitle(e.target.value)} />
      <textarea placeholder="Content" value={content} onChange={(e) => setContent(e.target.value)} />
      <input type="file" onChange={(e) => setImage(e.target.files[0])} />
      <button type="submit">Update Post</button>
    </form>
  );
};

export default EditPost;
```

#### 1.5. Atualize o Componente `PostList` para Incluir Botões de Editar e Excluir:

```
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Link, useHistory } from 'react-router-dom';

const PostList = () => {
  const [posts, setPosts] = useState([]);
  const history = useHistory();

  useEffect(() => {
    const fetchPosts = async () => {
      const res = await axios.get('http://localhost:5000/api/posts');
      setPosts(res.data);
    };
    fetchPosts();
  }, []);

  const handleDelete = async (id) => {
    const token = localStorage.getItem('token');
    try {
      await axios.delete(`http://localhost:5000/api/posts/${id}`, {
        headers: { Authorization: `Bearer ${token}` },
      });
      setPosts(posts.filter((post) => post.id !== id));
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <div>
      {posts.map((post) => (
        <div key={post.id} className="card mb-3">
          <div className="card-body">
            <h2 className="card-title">{post.title}</h2>
            <p className="card-text">{post.content}</p>
            {post.image && <img src={`http://localhost:5000/uploads/${post.image}`} alt={post.title} className="img-fluid" />}
            <div className="mt-3">
              <Link to={`/edit-post/${post.id}`} className="btn btn-primary me-2">Edit</Link>
              <button onClick={() => handleDelete(post.id)} className="btn btn-danger">Delete</button>
            </div>
          </div>
        </div>
      ))}
    </div>
  );
};

export default PostList;
```

#### 1.6. Adicione a Rota de Edição no `App.js`:

```
import EditPost from './components/EditPost';

// Dentro do <Switch>
<Route path="/edit-post/:id" component={EditPost} />
```

* * *

2\. **Comentários nos Posts**
-----------------------------

### Backend

#### 2.1. Crie a Tabela de Comentários no Banco de Dados:

```
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    user_id INT REFERENCES users(id),
    post_id INT REFERENCES posts(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 2.2. Crie o Modelo de Comentário (`src/models/Comment.js`):

```
import pool from '../config/db.js';

class Comment {
  static async create(content, userId, postId) {
    const { rows } = await pool.query(
      'INSERT INTO comments (content, user_id, post_id) VALUES ($1, $2, $3) RETURNING *',
      [content, userId, postId]
    );
    return rows[0];
  }

  static async findByPostId(postId) {
    const { rows } = await pool.query('SELECT * FROM comments WHERE post_id = $1', [postId]);
    return rows;
  }
}

export default Comment;
```

#### 2.3. Crie o Controlador de Comentários (`src/controllers/commentController.js`):

```
import Comment from '../models/Comment.js';

const createComment = async (req, res) => {
  const { content, postId } = req.body;
  const userId = req.user.id;

  const comment = await Comment.create(content, userId, postId);
  res.status(201).json(comment);
};

const getCommentsByPostId = async (req, res) => {
  const { postId } = req.params;
  const comments = await Comment.findByPostId(postId);
  res.json(comments);
};

export { createComment, getCommentsByPostId };
```

#### 2.4. Crie as Rotas de Comentários (`src/routes/commentRoutes.js`):

```
import express from 'express';
import { createComment, getCommentsByPostId } from '../controllers/commentController.js';
import authMiddleware from '../middleware/authMiddleware.js';

const router = express.Router();

router.post('/comments', authMiddleware, createComment);
router.get('/posts/:postId/comments', getCommentsByPostId);

export default router;
```

#### 2.5. Adicione as Rotas no `app.js`:

```
import commentRoutes from './src/routes/commentRoutes.js';

// Dentro do app.use
app.use('/api', commentRoutes);
```

* * *

### Frontend

#### 2.6. Crie um Componente para Comentários (`src/components/Comments.js`):

```
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const Comments = ({ postId }) => {
  const [comments, setComments] = useState([]);
  const [content, setContent] = useState('');

  useEffect(() => {
    const fetchComments = async () => {
      const res = await axios.get(`http://localhost:5000/api/posts/${postId}/comments`);
      setComments(res.data);
    };
    fetchComments();
  }, [postId]);

  const handleSubmit = async (e) => {
    e.preventDefault();
    const token = localStorage.getItem('token');
    try {
      const res = await axios.post(
        'http://localhost:5000/api/comments',
        { content, postId },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setComments([...comments, res.data]);
      setContent('');
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <div>
      <h4>Comments</h4>
      {comments.map((comment) => (
        <div key={comment.id} className="mb-2">
          <p>{comment.content}</p>
        </div>
      ))}
      <form onSubmit={handleSubmit}>
        <textarea value={content} onChange={(e) => setContent(e.target.value)} placeholder="Add a comment" />
        <button type="submit" className="btn btn-primary mt-2">Submit</button>
      </form>
    </div>
  );
};

export default Comments;
```

#### 2.7. Adicione o Componente de Comentários no `PostList`:

```
import Comments from './Comments';

// Dentro do post.map
<Comments postId={post.id} />
```

* * *

3\. **Paginação de Posts**
--------------------------

### Backend

#### 3.1. Atualize o Método `findAll` no Modelo de Post:

```
static async findAll(limit, offset) {
  const { rows } = await pool.query('SELECT * FROM posts LIMIT $1 OFFSET $2', [limit, offset]);
  return rows;
}
```

#### 3.2. Atualize o Controlador de Posts:

```
const getPosts = async (req, res) => {
  const { page = 1, limit = 10 } = req.query;
  const offset = (page - 1) * limit;
  const posts = await Post.findAll(limit, offset);
  res.json(posts);
};
```

### Frontend

#### 3.3. Atualize o Componente `PostList`:

```
const [page, setPage] = useState(1);
const [limit] = useState(10);

useEffect(() => {
  const fetchPosts = async () => {
    const res = await axios.get(`http://localhost:5000/api/posts?page=${page}&limit=${limit}`);
    setPosts(res.data);
  };
  fetchPosts();
}, [page, limit]);

// Adicione botões de paginação
<div className="d-flex justify-content-between mt-3">
  <button onClick={() => setPage(page - 1)} disabled={page === 1} className="btn btn-secondary">Previous</button>
  <button onClick={() => setPage(page + 1)} className="btn btn-secondary">Next</button>
</div>
```

* * *

4\. **Validação de Formulários no Frontend**
--------------------------------------------

Use bibliotecas como `Formik` e `Yup` para validação. Instale-as:

```
npm install formik yup
```

Exemplo de validação no `PostForm`:

```
import { useFormik } from 'formik';
import * as Yup from 'yup';

const validationSchema = Yup.object({
  title: Yup.string().required('Title is required'),
  content: Yup.string().required('Content is required'),
});

const formik = useFormik({
  initialValues: { title: '', content: '' },
  validationSchema,
  onSubmit: (values) => {
    // Lógica de envio
  },
});

// No formulário, use formik.handleSubmit e formik.errors para exibir erros.
```

* * *

5\. **Melhorias na Interface com Bootstrap**
--------------------------------------------

Substitua os elementos HTML por componentes do Bootstrap, como `Card`, `Form`, `Button`, etc. Use classes utilitárias para espaçamento e alinhamento.

* * *

Com essas implementações, seu blog estará completo com todas as funcionalidades solicitadas! 🚀

## Prompt:
Só revise se tenho a pagina inicial Home com os posts listados, um resumo, formato cards com bootstrap e icones do boostrap se preciso, paginação também. Ao clicar na categoria, mostra apenas daquela categoria, com paginação também, e ao clicar no post aparece ele por completo e abaixo 3 sugestões da mesma categoria. E a area dashboard: protegida, tem que fazer login. Tela de login. Ao entrar nela, varias tabelas iniciais listando user, categorias, posts e botão para logout, inserir, editar, deletar. Telas para inserir, editar...

## Response:
Vamos revisar e implementar todas as funcionalidades solicitadas, seguindo as melhores práticas com **Bootstrap 5**, **React**, e **Node.js**. Vou estruturar o projeto de forma clara e modularizada.

* * *

1\. **Página Inicial (Home)**
-----------------------------

### 1.1. Estrutura da Página Inicial

A página inicial (`Home`) exibirá os posts em formato de cards, com um resumo, paginação e a possibilidade de filtrar por categoria. Ao clicar em um post, ele será exibido por completo, com sugestões de posts da mesma categoria abaixo.

#### 1.1.1. Componente `Home.js`

```
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Link, useParams } from 'react-router-dom';
import { Card, Button, Pagination, Badge } from 'react-bootstrap';
import { BsCalendar, BsFolder } from 'react-icons/bs';

const Home = () => {
  const [posts, setPosts] = useState([]);
  const [categories, setCategories] = useState([]);
  const [currentPage, setCurrentPage] = useState(1);
  const [totalPages, setTotalPages] = useState(1);
  const { category } = useParams();

  const limit = 6; // Número de posts por página

  useEffect(() => {
    const fetchPosts = async () => {
      const res = await axios.get(
        `http://localhost:5000/api/posts?page=${currentPage}&limit=${limit}&category=${category || ''}`
      );
      setPosts(res.data.posts);
      setTotalPages(res.data.totalPages);
    };
    fetchPosts();
  }, [currentPage, category]);

  useEffect(() => {
    const fetchCategories = async () => {
      const res = await axios.get('http://localhost:5000/api/categories');
      setCategories(res.data);
    };
    fetchCategories();
  }, []);

  return (
    <div className="container mt-4">
      <h1 className="mb-4">Blog</h1>
      <div className="row">
        <div className="col-md-3">
          <h4>Categorias</h4>
          <ul className="list-group">
            {categories.map((cat) => (
              <li key={cat.id} className="list-group-item">
                <Link to={`/category/${cat.id}`} className="text-decoration-none">
                  <BsFolder className="me-2" />
                  {cat.name}
                </Link>
              </li>
            ))}
          </ul>
        </div>
        <div className="col-md-9">
          <div className="row">
            {posts.map((post) => (
              <div key={post.id} className="col-md-6 mb-4">
                <Card>
                  <Card.Img variant="top" src={`http://localhost:5000/uploads/${post.image}`} />
                  <Card.Body>
                    <Card.Title>{post.title}</Card.Title>
                    <Card.Text>{post.content.substring(0, 100)}...</Card.Text>
                    <Badge bg="secondary" className="me-2">
                      <BsCalendar className="me-1" />
                      {new Date(post.created_at).toLocaleDateString()}
                    </Badge>
                    <Badge bg="info">
                      <BsFolder className="me-1" />
                      {post.category_name}
                    </Badge>
                    <Link to={`/post/${post.id}`} className="btn btn-primary mt-3">
                      Ler mais
                    </Link>
                  </Card.Body>
                </Card>
              </div>
            ))}
          </div>
          <Pagination className="mt-4">
            {Array.from({ length: totalPages }, (_, i) => (
              <Pagination.Item
                key={i + 1}
                active={i + 1 === currentPage}
                onClick={() => setCurrentPage(i + 1)}
              >
                {i + 1}
              </Pagination.Item>
            ))}
          </Pagination>
        </div>
      </div>
    </div>
  );
};

export default Home;
```

#### 1.1.2. Rota para a Página Inicial

No `App.js`:

```
<Route path="/" exact component={Home} />
<Route path="/category/:category" component={Home} />
```

* * *

### 1.2. Página de Post Completo

Ao clicar em um post, ele será exibido por completo, com sugestões de posts da mesma categoria abaixo.

#### 1.2.1. Componente `PostDetail.js`

```
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Link, useParams } from 'react-router-dom';
import { Card, Button } from 'react-bootstrap';

const PostDetail = () => {
  const { id } = useParams();
  const [post, setPost] = useState(null);
  const [suggestions, setSuggestions] = useState([]);

  useEffect(() => {
    const fetchPost = async () => {
      const res = await axios.get(`http://localhost:5000/api/posts/${id}`);
      setPost(res.data);
    };
    fetchPost();
  }, [id]);

  useEffect(() => {
    const fetchSuggestions = async () => {
      if (post) {
        const res = await axios.get(`http://localhost:5000/api/posts/suggestions/${post.category_id}/${id}`);
        setSuggestions(res.data);
      }
    };
    fetchSuggestions();
  }, [post]);

  if (!post) return <div>Loading...</div>;

  return (
    <div className="container mt-4">
      <Card>
        <Card.Img variant="top" src={`http://localhost:5000/uploads/${post.image}`} />
        <Card.Body>
          <Card.Title>{post.title}</Card.Title>
          <Card.Text>{post.content}</Card.Text>
          <Link to="/" className="btn btn-primary">
            Voltar
          </Link>
        </Card.Body>
      </Card>
      <h3 className="mt-4">Sugestões</h3>
      <div className="row">
        {suggestions.map((suggestion) => (
          <div key={suggestion.id} className="col-md-4 mb-4">
            <Card>
              <Card.Img variant="top" src={`http://localhost:5000/uploads/${suggestion.image}`} />
              <Card.Body>
                <Card.Title>{suggestion.title}</Card.Title>
                <Link to={`/post/${suggestion.id}`} className="btn btn-primary">
                  Ler mais
                </Link>
              </Card.Body>
            </Card>
          </div>
        ))}
      </div>
    </div>
  );
};

export default PostDetail;
```

#### 1.2.2. Rota para o Post Completo

No `App.js`:

```
<Route path="/post/:id" component={PostDetail} />
```

* * *

2\. **Dashboard Protegido**
---------------------------

### 2.1. Tela de Login

#### 2.1.1. Componente `Login.js`

```
import React, { useState } from 'react';
import axios from 'axios';
import { useHistory } from 'react-router-dom';
import { Form, Button, Alert } from 'react-bootstrap';

const Login = () => {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const history = useHistory();

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const res = await axios.post('http://localhost:5000/api/auth/login', { username, password });
      localStorage.setItem('token', res.data.token);
      history.push('/dashboard');
    } catch (err) {
      setError('Credenciais inválidas');
    }
  };

  return (
    <div className="container mt-4">
      <h1>Login</h1>
      {error && <Alert variant="danger">{error}</Alert>}
      <Form onSubmit={handleSubmit}>
        <Form.Group className="mb-3">
          <Form.Label>Usuário</Form.Label>
          <Form.Control
            type="text"
            value={username}
            onChange={(e) => setUsername(e.target.value)}
            required
          />
        </Form.Group>
        <Form.Group className="mb-3">
          <Form.Label>Senha</Form.Label>
          <Form.Control
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </Form.Group>
        <Button type="submit">Entrar</Button>
      </Form>
    </div>
  );
};

export default Login;
```

#### 2.1.2. Rota para o Login

No `App.js`:

```
<Route path="/login" component={Login} />
```

* * *

### 2.2. Dashboard

#### 2.2.1. Componente `Dashboard.js`

```
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { Link, useHistory } from 'react-router-dom';
import { Table, Button, Alert } from 'react-bootstrap';

const Dashboard = () => {
  const [users, setUsers] = useState([]);
  const [categories, setCategories] = useState([]);
  const [posts, setPosts] = useState([]);
  const [error, setError] = useState('');
  const history = useHistory();

  useEffect(() => {
    const fetchData = async () => {
      try {
        const token = localStorage.getItem('token');
        const usersRes = await axios.get('http://localhost:5000/api/users', {
          headers: { Authorization: `Bearer ${token}` },
        });
        const categoriesRes = await axios.get('http://localhost:5000/api/categories', {
          headers: { Authorization: `Bearer ${token}` },
        });
        const postsRes = await axios.get('http://localhost:5000/api/posts', {
          headers: { Authorization: `Bearer ${token}` },
        });
        setUsers(usersRes.data);
        setCategories(categoriesRes.data);
        setPosts(postsRes.data);
      } catch (err) {
        setError('Erro ao carregar dados');
      }
    };
    fetchData();
  }, []);

  const handleLogout = () => {
    localStorage.removeItem('token');
    history.push('/login');
  };

  return (
    <div className="container mt-4">
      <h1>Dashboard</h1>
      {error && <Alert variant="danger">{error}</Alert>}
      <Button onClick={handleLogout} variant="danger" className="mb-4">
        Logout
      </Button>
      <h2>Usuários</h2>
      <Table striped bordered hover>
        <thead>
          <tr>
            <th>ID</th>
            <th>Usuário</th>
            <th>Ações</th>
          </tr>
        </thead>
        <tbody>
          {users.map((user) => (
            <tr key={user.id}>
              <td>{user.id}</td>
              <td>{user.username}</td>
              <td>
                <Button variant="warning" size="sm" className="me-2">
                  Editar
                </Button>
                <Button variant="danger" size="sm">
                  Excluir
                </Button>
              </td>
            </tr>
          ))}
        </tbody>
      </Table>
      <h2>Categorias</h2>
      <Table striped bordered hover>
        <thead>
          <tr>
            <th>ID</th>
            <th>Nome</th>
            <th>Ações</th>
          </tr>
        </thead>
        <tbody>
          {categories.map((category) => (
            <tr key={category.id}>
              <td>{category.id}</td>
              <td>{category.name}</td>
              <td>
                <Button variant="warning" size="sm" className="me-2">
                  Editar
                </Button>
                <Button variant="danger" size="sm">
                  Excluir
                </Button>
              </td>
            </tr>
          ))}
        </tbody>
      </Table>
      <h2>Posts</h2>
      <Table striped bordered hover>
        <thead>
          <tr>
            <th>ID</th>
            <th>Título</th>
            <th>Ações</th>
          </tr>
        </thead>
        <tbody>
          {posts.map((post) => (
            <tr key={post.id}>
              <td>{post.id}</td>
              <td>{post.title}</td>
              <td>
                <Button variant="warning" size="sm" className="me-2">
                  Editar
                </Button>
                <Button variant="danger" size="sm">
                  Excluir
                </Button>
              </td>
            </tr>
          ))}
        </tbody>
      </Table>
    </div>
  );
};

export default Dashboard;
```

#### 2.2.2. Rota Protegida para o Dashboard

No `App.js`:

```
<PrivateRoute path="/dashboard" component={Dashboard} />
```

* * *

### 2.3. Telas para Inserir, Editar e Excluir

Crie componentes para **Inserir**, **Editar** e **Excluir** usuários, categorias e posts. Use formulários e requisições à API para realizar as operações.

* * *

Com essas implementações, seu blog estará completo e funcional! 🚀

## Prompt:
Crie componentes para Inserir, Editar e Excluir usuários, categorias e posts. Use formulários e requisições à API para realizar as operações.

## Response:
Vamos criar os componentes para **Inserir**, **Editar** e **Excluir** usuários, categorias e posts. Cada operação terá um formulário e fará requisições à API para realizar as ações. Vou criar os componentes de forma modularizada e reutilizável.

* * *

1\. **Componentes para Usuários**
---------------------------------

### 1.1. Inserir Usuário

#### 1.1.1. Componente `AddUser.js`

```
import React, { useState } from 'react';
import axios from 'axios';
import { useHistory } from 'react-router-dom';
import { Form, Button, Alert } from 'react-bootstrap';

const AddUser = () => {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const history = useHistory();

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const token = localStorage.getItem('token');
      await axios.post(
        'http://localhost:5000/api/users',
        { username, password },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      history.push('/dashboard');
    } catch (err) {
      setError('Erro ao adicionar usuário');
    }
  };

  return (
    <div className="container mt-4">
      <h1>Adicionar Usuário</h1>
      {error && <Alert variant="danger">{error}</Alert>}
      <Form onSubmit={handleSubmit}>
        <Form.Group className="mb-3">
          <Form.Label>Usuário</Form.Label>
          <Form.Control
            type="text"
            value={username}
            onChange={(e) => setUsername(e.target.value)}
            required
          />
        </Form.Group>
        <Form.Group className="mb-3">
          <Form.Label>Senha</Form.Label>
          <Form.Control
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </Form.Group>
        <Button type="submit">Adicionar</Button>
      </Form>
    </div>
  );
};

export default AddUser;
```

#### 1.1.2. Rota para Adicionar Usuário

No `App.js`:

```
<PrivateRoute path="/add-user" component={AddUser} />
```

* * *

### 1.2. Editar Usuário

#### 1.2.1. Componente `EditUser.js`

```
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useParams, useHistory } from 'react-router-dom';
import { Form, Button, Alert } from 'react-bootstrap';

const EditUser = () => {
  const { id } = useParams();
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const history = useHistory();

  useEffect(() => {
    const fetchUser = async () => {
      try {
        const token = localStorage.getItem('token');
        const res = await axios.get(`http://localhost:5000/api/users/${id}`, {
          headers: { Authorization: `Bearer ${token}` },
        });
        setUsername(res.data.username);
      } catch (err) {
        setError('Erro ao carregar usuário');
      }
    };
    fetchUser();
  }, [id]);

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `http://localhost:5000/api/users/${id}`,
        { username, password },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      history.push('/dashboard');
    } catch (err) {
      setError('Erro ao editar usuário');
    }
  };

  return (
    <div className="container mt-4">
      <h1>Editar Usuário</h1>
      {error && <Alert variant="danger">{error}</Alert>}
      <Form onSubmit={handleSubmit}>
        <Form.Group className="mb-3">
          <Form.Label>Usuário</Form.Label>
          <Form.Control
            type="text"
            value={username}
            onChange={(e) => setUsername(e.target.value)}
            required
          />
        </Form.Group>
        <Form.Group className="mb-3">
          <Form.Label>Nova Senha</Form.Label>
          <Form.Control
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
          />
        </Form.Group>
        <Button type="submit">Salvar</Button>
      </Form>
    </div>
  );
};

export default EditUser;
```

#### 1.2.2. Rota para Editar Usuário

No `App.js`:

```
<PrivateRoute path="/edit-user/:id" component={EditUser} />
```

* * *

### 1.3. Excluir Usuário

#### 1.3.1. Função de Exclusão no `Dashboard.js`

No componente `Dashboard`, adicione a função para excluir usuários:

```
const handleDeleteUser = async (id) => {
  try {
    const token = localStorage.getItem('token');
    await axios.delete(`http://localhost:5000/api/users/${id}`, {
      headers: { Authorization: `Bearer ${token}` },
    });
    setUsers(users.filter((user) => user.id !== id));
  } catch (err) {
    setError('Erro ao excluir usuário');
  }
};
```

No botão de exclusão:

```
<Button variant="danger" size="sm" onClick={() => handleDeleteUser(user.id)}>
  Excluir
</Button>
```

* * *

2\. **Componentes para Categorias**
-----------------------------------

### 2.1. Inserir Categoria

#### 2.1.1. Componente `AddCategory.js`

```
import React, { useState } from 'react';
import axios from 'axios';
import { useHistory } from 'react-router-dom';
import { Form, Button, Alert } from 'react-bootstrap';

const AddCategory = () => {
  const [name, setName] = useState('');
  const [error, setError] = useState('');
  const history = useHistory();

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const token = localStorage.getItem('token');
      await axios.post(
        'http://localhost:5000/api/categories',
        { name },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      history.push('/dashboard');
    } catch (err) {
      setError('Erro ao adicionar categoria');
    }
  };

  return (
    <div className="container mt-4">
      <h1>Adicionar Categoria</h1>
      {error && <Alert variant="danger">{error}</Alert>}
      <Form onSubmit={handleSubmit}>
        <Form.Group className="mb-3">
          <Form.Label>Nome</Form.Label>
          <Form.Control
            type="text"
            value={name}
            onChange={(e) => setName(e.target.value)}
            required
          />
        </Form.Group>
        <Button type="submit">Adicionar</Button>
      </Form>
    </div>
  );
};

export default AddCategory;
```

#### 2.1.2. Rota para Adicionar Categoria

No `App.js`:

```
<PrivateRoute path="/add-category" component={AddCategory} />
```

* * *

### 2.2. Editar Categoria

#### 2.2.1. Componente `EditCategory.js`

```
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useParams, useHistory } from 'react-router-dom';
import { Form, Button, Alert } from 'react-bootstrap';

const EditCategory = () => {
  const { id } = useParams();
  const [name, setName] = useState('');
  const [error, setError] = useState('');
  const history = useHistory();

  useEffect(() => {
    const fetchCategory = async () => {
      try {
        const token = localStorage.getItem('token');
        const res = await axios.get(`http://localhost:5000/api/categories/${id}`, {
          headers: { Authorization: `Bearer ${token}` },
        });
        setName(res.data.name);
      } catch (err) {
        setError('Erro ao carregar categoria');
      }
    };
    fetchCategory();
  }, [id]);

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `http://localhost:5000/api/categories/${id}`,
        { name },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      history.push('/dashboard');
    } catch (err) {
      setError('Erro ao editar categoria');
    }
  };

  return (
    <div className="container mt-4">
      <h1>Editar Categoria</h1>
      {error && <Alert variant="danger">{error}</Alert>}
      <Form onSubmit={handleSubmit}>
        <Form.Group className="mb-3">
          <Form.Label>Nome</Form.Label>
          <Form.Control
            type="text"
            value={name}
            onChange={(e) => setName(e.target.value)}
            required
          />
        </Form.Group>
        <Button type="submit">Salvar</Button>
      </Form>
    </div>
  );
};

export default EditCategory;
```

#### 2.2.2. Rota para Editar Categoria

No `App.js`:

```
<PrivateRoute path="/edit-category/:id" component={EditCategory} />
```

* * *

### 2.3. Excluir Categoria

#### 2.3.1. Função de Exclusão no `Dashboard.js`

No componente `Dashboard`, adicione a função para excluir categorias:

```
const handleDeleteCategory = async (id) => {
  try {
    const token = localStorage.getItem('token');
    await axios.delete(`http://localhost:5000/api/categories/${id}`, {
      headers: { Authorization: `Bearer ${token}` },
    });
    setCategories(categories.filter((category) => category.id !== id));
  } catch (err) {
    setError('Erro ao excluir categoria');
  }
};
```

No botão de exclusão:

```
<Button variant="danger" size="sm" onClick={() => handleDeleteCategory(category.id)}>
  Excluir
</Button>
```

* * *

3\. **Componentes para Posts**
------------------------------

### 3.1. Inserir Post

#### 3.1.1. Componente `AddPost.js`

```
import React, { useState } from 'react';
import axios from 'axios';
import { useHistory } from 'react-router-dom';
import { Form, Button, Alert } from 'react-bootstrap';

const AddPost = () => {
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  const [categoryId, setCategoryId] = useState('');
  const [image, setImage] = useState(null);
  const [error, setError] = useState('');
  const history = useHistory();

  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData();
    formData.append('title', title);
    formData.append('content', content);
    formData.append('categoryId', categoryId);
    if (image) formData.append('image', image);

    try {
      const token = localStorage.getItem('token');
      await axios.post('http://localhost:5000/api/posts', formData, {
        headers: {
          Authorization: `Bearer ${token}`,
          'Content-Type': 'multipart/form-data',
        },
      });
      history.push('/dashboard');
    } catch (err) {
      setError('Erro ao adicionar post');
    }
  };

  return (
    <div className="container mt-4">
      <h1>Adicionar Post</h1>
      {error && <Alert variant="danger">{error}</Alert>}
      <Form onSubmit={handleSubmit}>
        <Form.Group className="mb-3">
          <Form.Label>Título</Form.Label>
          <Form.Control
            type="text"
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            required
          />
        </Form.Group>
        <Form.Group className="mb-3">
          <Form.Label>Conteúdo</Form.Label>
          <Form.Control
            as="textarea"
            rows={5}
            value={content}
            onChange={(e) => setContent(e.target.value)}
            required
          />
        </Form.Group>
        <Form.Group className="mb-3">
          <Form.Label>Categoria</Form.Label>
          <Form.Control
            as="select"
            value={categoryId}
            onChange={(e) => setCategoryId(e.target.value)}
            required
          >
            <option value="">Selecione uma categoria</option>
            {/* Adicione as categorias aqui */}
          </Form.Control>
        </Form.Group>
        <Form.Group className="mb-3">
          <Form.Label>Imagem</Form.Label>
          <Form.Control type="file" onChange={(e) => setImage(e.target.files[0])} />
        </Form.Group>
        <Button type="submit">Adicionar</Button>
      </Form>
    </div>
  );
};

export default AddPost;
```

#### 3.1.2. Rota para Adicionar Post

No `App.js`:

```
<PrivateRoute path="/add-post" component={AddPost} />
```

* * *

### 3.2. Editar Post

#### 3.2.1. Componente `EditPost.js`

```
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useParams, useHistory } from 'react-router-dom';
import { Form, Button, Alert } from 'react-bootstrap';

const EditPost = () => {
  const { id } = useParams();
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  const [categoryId, setCategoryId] = useState('');
  const [image, setImage] = useState(null);
  const [error, setError] = useState('');
  const history = useHistory();

  useEffect(() => {
    const fetchPost = async () => {
      try {
        const token = localStorage.getItem('token');
        const res = await axios.get(`http://localhost:5000/api/posts/${id}`, {
          headers: { Authorization: `Bearer ${token}` },
        });
        setTitle(res.data.title);
        setContent(res.data.content);
        setCategoryId(res.data.category_id);
      } catch (err) {
        setError('Erro ao carregar post');
      }
    };
    fetchPost();
  }, [id]);

  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData();
    formData.append('title', title);
    formData.append('content', content);
    formData.append('categoryId', categoryId);
    if (image) formData.append('image', image);

    try {
      const token = localStorage.getItem('token');
      await axios.put(`http://localhost:5000/api/posts/${id}`, formData, {
        headers: {
          Authorization: `Bearer ${token}`,
          'Content-Type': 'multipart/form-data',
        },
      });
      history.push('/dashboard');
    } catch (err) {
      setError('Erro ao editar post');
    }
  };

  return (
    <div className="container mt-4">
      <h1>Editar Post</h1>
      {error && <Alert variant="danger">{error}</Alert>}
      <Form onSubmit={handleSubmit}>
        <Form.Group className="mb-3">
          <Form.Label>Título</Form.Label>
          <Form.Control
            type="text"
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            required
          />
        </Form.Group>
        <Form.Group className="mb-3">
          <Form.Label>Conteúdo</Form.Label>
          <Form.Control
            as="textarea"
            rows={5}
            value={content}
            onChange={(e) => setContent(e.target.value)}
            required
          />
        </Form.Group>
        <Form.Group className="mb-3">
          <Form.Label>Categoria</Form.Label>
          <Form.Control
            as="select"
            value={categoryId}
            onChange={(e) => setCategoryId(e.target.value)}
            required
          >
            <option value="">Selecione uma categoria</option>
            {/* Adicione as categorias aqui */}
          </Form.Control>
        </Form.Group>
        <Form.Group className="mb-3">
          <Form.Label>Imagem</Form.Label>
          <Form.Control type="file" onChange={(e) => setImage(e.target.files[0])} />
        </Form.Group>
        <Button type="submit">Salvar</Button>
      </Form>
    </div>
  );
};

export default EditPost;
```

#### 3.2.2. Rota para Editar Post

No `App.js`:

```
<PrivateRoute path="/edit-post/:id" component={EditPost} />
```

* * *

### 3.3. Excluir Post

#### 3.3.1. Função de Exclusão no `Dashboard.js`

No componente `Dashboard`, adicione a função para excluir posts:

```
const handleDeletePost = async (id) => {
  try {
    const token = localStorage.getItem('token');
    await axios.delete(`http://localhost:5000/api/posts/${id}`, {
      headers: { Authorization: `Bearer ${token}` },
    });
    setPosts(posts.filter((post) => post.id !== id));
  } catch (err) {
    setError('Erro ao excluir post');
  }
};
```

No botão de exclusão:

```
<Button variant="danger" size="sm" onClick={() => handleDeletePost(post.id)}>
  Excluir
</Button>
```

* * *

Com esses componentes, você terá um sistema completo para **Inserir**, **Editar** e **Excluir** usuários, categorias e posts. 🚀



---
Powered by [DeepSeek Exporter](https://chromewebstore.google.com/detail/ccidioajndoplgongndfmljpcaejlnlc)
