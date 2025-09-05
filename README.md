# shadow
```python name=flask_tweet_app.py
# Instale as bibliotecas necessárias para rodar o app no Google Colab
!pip install Flask==2.3.2 Pillow==9.5.0 pyngrok==6.0.0

from flask import Flask, render_template, request, jsonify, redirect, url_for
from datetime import datetime
import os
from werkzeug.utils import secure_filename
from PIL import Image
from pyngrok import ngrok
import threading
import time
import random

# Use a porta 5000, que é a padrão do Flask
PORT = 5000

# Crie a instância do Flask
app = Flask(__name__, template_folder='templates', static_folder='static')

# Configuração do Flask
UPLOAD_FOLDER = 'static/uploads'
ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif'}
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
app.config['MAX_CONTENT_LENGTH'] = 5 * 1024 * 1024  # Max 5MB file size

# Garante que a pasta de uploads e a pasta de templates existam
os.makedirs('templates', exist_ok=True)
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

# Armazenamento de tweets em memória
tweets = [
    {
        'id': 1,
        'text': "A verdadeira liberdade de expressão só existe quando podemos falar sem medo de julgamento ou retaliação.",
        'timestamp': datetime.now().timestamp() - 300,
        'likes': 23,
        'retweets': 5,
        'replies': 8,
        'liked': False,
        'retweeted': False,
        'is_own': False,
        'liked_by': ['@anon15', '@anon32', '@anon7', '@anon91', '@anon44'],
        'image': None
    },
    {
        'id': 2,
        'text': "Privacidade não é sobre esconder algo errado. É sobre proteger o que é certo.",
        'timestamp': datetime.now().timestamp() - 600,
        'likes': 156,
        'retweets': 42,
        'replies': 31,
        'liked': False,
        'retweeted': False,
        'is_own': False,
        'liked_by': ['@anon23', '@anon88', '@anon12', '@anon67', '@anon99'],
        'image': None
    },
    {
        'id': 3,
        'text': "Em um mundo onde cada clique é rastreado, o anonimato se torna um ato de resistência.",
        'timestamp': datetime.now().timestamp() - 900,
        'likes': 89,
        'retweets': 12,
        'replies': 15,
        'liked': False,
        'retweeted': False,
        'is_own': False,
        'liked_by': ['@anon56', '@anon11', '@anon78', '@anon33', '@anon92'],
        'image': None
    }
]

tweet_id_counter = 4
random_tweets = [
    "O anonimato permite que as ideias sejam julgadas pelo seu mérito, não pela fonte.",
    "Cada pessoa deveria ter o direito de expressar suas opiniões sem medo de perseguição.",
    "A transparência do governo é essencial, mas a privacidade do cidadão também é.",
    "Questionar é um direito fundamental. O anonimato protege esse direito.",
    "Nem toda verdade precisa de um rosto para ser válida."
]

def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

def format_time(timestamp):
    now = datetime.now().timestamp()
    diff = now - timestamp
    minutes = int(diff // 60)
    hours = int(diff // 3600)
    days = int(diff // 86400)

    if minutes < 1:
        return 'agora'
    if minutes < 60:
        return f'{minutes}m'
    if hours < 24:
        return f'{hours}h'
    return f'{days}d'

@app.route('/')
def index():
    return render_template('index.html', tweets=tweets, format_time=format_time)

@app.route('/post_tweet', methods=['POST'])
def post_tweet():
    global tweet_id_counter
    tweet_text = request.form.get('tweetText', '').strip()
    image_file = request.files.get('imageInput')

    image_path = None
    if image_file and allowed_file(image_file.filename):
        # Validate file size
        image_file.seek(0, os.SEEK_END)
        if image_file.tell() > app.config['MAX_CONTENT_LENGTH']:
            return jsonify({'error': 'Arquivo muito grande! Máximo 5MB.'}), 400
        image_file.seek(0)

        filename = secure_filename(f"{datetime.now().timestamp()}_{image_file.filename}")
        image_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)
        image_file.save(image_path)

        # Optimize image (resize to max 800px width/height)
        with Image.open(image_path) as img:
            img.thumbnail((800, 800))
            img.save(image_path)
            
        # Gera o caminho da imagem para ser acessível na web
        image_path = f"/{image_path}"

    if not tweet_text and not image_path:
        return jsonify({'error': 'Texto ou imagem necessários.'}), 400

    if len(tweet_text) > 500:
        return jsonify({'error': 'Texto excede 500 caracteres.'}), 400

    new_tweet = {
        'id': tweet_id_counter,
        'text': tweet_text,
        'timestamp': datetime.now().timestamp(),
        'likes': 0,
        'retweets': 0,
        'replies': 0,
        'liked': False,
        'retweeted': False,
        'is_own': True,
        'liked_by': [],
        'image': image_path
    }

    tweets.insert(0, new_tweet)
    tweet_id_counter += 1
    return jsonify({'success': True})

@app.route('/toggle_like/<int:tweet_id>', methods=['POST'])
def toggle_like(tweet_id):
    tweet = next((t for t in tweets if t['id'] == tweet_id), None)
    if not tweet:
        return jsonify({'error': 'Tweet não encontrado.'}), 404

    user_handle = f"@anon{random.randint(1, 999)}"
    tweet['liked'] = not tweet['liked']
    if tweet['liked']:
        tweet['likes'] += 1
        tweet['liked_by'].append(user_handle)
    else:
        tweet['likes'] -= 1
        if user_handle in tweet['liked_by']:
            tweet['liked_by'].remove(user_handle)

    return jsonify({'success': True, 'likes': tweet['likes'], 'liked': tweet['liked']})

@app.route('/toggle_reply/<int:tweet_id>', methods=['POST'])
def toggle_reply(tweet_id):
    tweet = next((t for t in tweets if t['id'] == tweet_id), None)
    if not tweet:
        return jsonify({'error': 'Tweet não encontrado.'}), 404

    tweet['replies'] += 1
    return jsonify({'success': True, 'replies': tweet['replies']})

@app.route('/get_likes/<int:tweet_id>', methods=['GET'])
def get_likes(tweet_id):
    tweet = next((t for t in tweets if t['id'] == tweet_id), None)
    if not tweet:
        return jsonify({'error': 'Tweet não encontrado.'}), 404

    if tweet['liked_by']:
        likes_text = ', '.join(tweet['liked_by'][:10])
        more_text = f" e mais {len(tweet['liked_by']) - 10} usuários" if len(tweet['liked_by']) > 10 else ''
        return jsonify({'likes': f"Curtido por: {likes_text}{more_text}"})
    return jsonify({'likes': 'Nenhuma curtida ainda.'})

def add_random_tweets():
    while True:
        if random.random() < 0.3:
            global tweet_id_counter
            random_text = random.choice(random_tweets)
            new_tweet = {
                'id': tweet_id_counter,
                'text': random_text,
                'timestamp': datetime.now().timestamp(),
                'likes': random.randint(0, 50),
                'retweets': random.randint(0, 20),
                'replies': random.randint(0, 15),
                'liked': False,
                'retweeted': False,
                'is_own': False,
                'liked_by': [],
                'image': None
            }
            tweets.insert(0, new_tweet)
            tweet_id_counter += 1
            if len(tweets) > 20:
                tweets.pop()
        time.sleep(45)

threading.Thread(target=add_random_tweets, daemon=True).start()

# Inicia o túnel ngrok e a aplicação Flask
def run_flask_app():
    try:
        ngrok.set_auth_token("YOUR_AUTH_TOKEN") # Substitua por seu token de autenticação
        public_url = ngrok.connect(PORT).public_url
        print(f"Flask App rodando em: {public_url}")
        app.run(port=PORT, debug=False, use_reloader=False)
    except Exception as e:
        print(f"Erro ao iniciar a aplicação: {e}")
    finally:
        ngrok.disconnect()

if __name__ == '__main__':
    run_flask_app()
```
