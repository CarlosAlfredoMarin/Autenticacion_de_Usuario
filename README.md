# Inicio de Sesión con Python, Flask y Docker

En este documento se explican uno a uno todos los archivos utilizados en el proyecto, la explicación se realiza línea por línea.  
<br><br>

## **Debugging:**
 Es el nombre que se le da al proceso de encontrar y eliminar los errores que pueden cometer softwares y hardwares.  
<br><br>








# main.py
La idea es configurar la aplicación en una función. Creo una instancia de la clase ```create_app```
```python
app = create_app()
```  
<br>


## **Error Handlers:** 
Un controlador de errores es una función que devuelve una respuesta cuando se genera un tipo de error.

## **Renderizar:**
El renderizado es el proceso de mostrar tu página web en un navegador.  
<br>

Cuando ocurre un error ```404``` se renderiza la plantilla html que preparamos.

```python
@app.errorhandler(404)
def not_found(error):
    return render_template('404.html', error=error)
```

Cuando ocurre un error ```500``` se renderiza la plantilla html que preparamos.
```python
@app.errorhandler(500)
def server_error(error):
    return render_template('500.html', error=error)
```

Obtener la IP del usuario que está ingresando a nuestra aplicación
```python
@app.route('/')
def index():
    user_ip = request.remote_addr
```

Me redirige a la ruta ```hello``
```python    
    response = make_response(redirect('/hello'))
    session['user_ip'] = user_ip
    return response
```

```session``` se utiliza para guardar a través de varias peticiones información del usuario de manera segura (encriptada)
```python
@app.route('/hello', methods=['GET'])
@login_required
def hello():
    user_ip = session.get('user_ip')
    username = session.get('username')
```

El contexto de la aplicación es un buen lugar para almacenar datos comunes durante una solicitud
```python
    context = {
        'user_ip': user_ip, 
        'todos': todos,
        'username': username
    }
```

Renderiza la plantilla ```hello.html```    
```python    
    return render_template('hello.html', **context)
```
<br><br><br><br>









# app/\_\_Init\_\_.py
Importamos las librerías que se utilizan en este mismo archivo
```python
from flask import Flask
from flask_bootstrap import Bootstrap
from flask_login import LoginManager
from .config import Config
from .auth import auth
from .models import  UserModel
```
<br><br>

Creamos un objeto de la clase ```LoginManager```, ya que, utilizaremos varios métodos de esta clase.
```python
login_manager = LoginManager()
```

El nombre de la vista a la que se redirigirá cuando el usuario necesite iniciar sesión.
```python
login_manager.login_view = 'auth.login'
```
<br><br>

La función ```create_app() ``` es la que nos va a regresar la nueva aplicación

```python
def create_app():
```

La variable ```__name__``` se pasa como primer argumento al crear una instancia del objeto Flask (una aplicación Python Flask). En este caso, ```__name__``` representa el nombre del paquete de la aplicación.

```python
    app = Flask(__name__)
```

Creamos un objeto de la clase ```Bootstrap```
```python
    bootstrap = Bootstrap(app)
```

La configuración se vuelve más útil si puedo almacenarla en un archivo separado, idealmente ubicado fuera del paquete de la aplicación real.

Esta línea carga la configuración de ```config.py```
```python    
    app.config.from_object(Config)
```

Inicializacmos la aplicación
```python 
    login_manager.init_app(app)
```

Registro de ```blueprint```, un ```blueprint``` sirve para organizar un grupo de vistas
```python  
    app.register_blueprint(auth)
    return app
```
<br><br><br><br>









# forms.py
Importamos las librerías que se utilizan en este mismo archivo
```python
from flask_wtf import FlaskForm
from wtforms.fields import StringField, PasswordField, SubmitField
from wtforms.validators import DataRequired
```
<br><br>


Formulario de registro, para que los usuarios puedan registrarse a través de un formulario web
```python
class LoginForm(FlaskForm):
    username = StringField('Nombre de usuario', validators=[DataRequired()])
    password = PasswordField('Password', validators=[DataRequired()])
    submit = SubmitField('Enviar')
```    
<br><br><br><br>









# auth/\_\_Init\_\_.py
Creamos un ```blueprint```, el cual va a tener el nombre ```auth```. Todas las rutas que comiencen con ```/auth``` van a ser rutedas o dirigidas a este ```blueprint```
```python
from flask import Blueprint

auth = Blueprint('auth', __name__, url_prefix='/auth')
``` 
<br><br><br><br>









# auth/views.py

Importamos las librerías que se utilizan en este mismo archivo
```python
from flask import render_template, session, redirect, flash, url_for
from . import auth
from app.forms import LoginForm
``` 

En esta ruta, voy a utilizar 2 métodos, uno para publicar las credenciales de usuario en la Base de Datos y otro para extraer las credenciales de la Base de Datos.
```python
@auth.route('/login', methods=['GET', 'POST'])
```

Creo una instancia de la clase LoginForm() 
```python
def login():
    login_form = LoginForm()
```    

```python
    context = {
        'login_form': login_form
    }
```  

El método ```validate_on_submit()``` del formulario devuelve ```True``` cuando se envió el formulario y todos los validadores de campo aceptaron los datos. En todos los demás casos, ```validate_on_submit()``` devuelve False.    
```python
    if login_form.validate_on_submit():
        username = login_form.username.data
        session['username'] = username
        flash('Nombre de usuario registrado con éxito')
        return redirect(url_for('index'))

    return render_template('login.html', **context)
``` 
<br><br><br><br>









# models.py

Importamos las librerías que se utilizan en este mismo archivo. ```UserMixin``` ya trae implementadas las propiedades que necesita ```flask_login```, entonces ya no es necesario que nosotros las implementemos
```python
from flask_login import UserMixin
from firestore_service import get_user
```

Paraa segurarnos que siempre tenemos la información que requerimos
```python
class UserData:
    def __init__(self, username, passsword) -> None:
        self.username = username
        self.passsword = passsword
```

```python
class UserModel(UserMixin):
    def __init__(self, user_data) -> None:
        self.id = user_data.username
        self.password = user_data.password
```

Cada vez que flask-login quiera cargar el usuario vamos a llamar esta ```query```. En la función se busca el usuario en la base de datos y regresarlo
```python
    @staticmethod
    def query(user_id):
        user_doc = get_user(user_id)
        user_data = UserData(
            username = user_doc.id,
            password = user_doc.to_dict()['password']
        )
        return UserModel(user_data)
```








# firestore_service.py

Importamos la librería para utilizar Google Cloud
```python
import firebase_admin
from firebase_admin import credentials
from firebase_admin import firestore
```

Indicamos el ID de la Base de Datos creada en Google Cloud. Creamos un objeto ```db`` que será el cliente de la base de datos.
```python
project_id = 'my-project-flask-358422'
credential = credentials.ApplicationDefault()
firebase_admin.initialize_app(credential, {
  'projectId': project_id,
})
db = firestore.client()
```

Leer todos los usuarios de la db
```python
def get_users():
    return db.collection('users').get()
```

Obtener un usuario de la db
```python
def get_user(user_id):
    return db.collection('users').document(user_id).get()
```

Registrar un usuario con contraseña en la db
```python
def user_put(user_data):
    user_ref = db.collection('users').document(user_data.username)
    user_ref.set({'password': user_data.password})
```

Obtener el listado de tareas correspondiente a un usuario 
```python
def get_todos(user_id):
    return db.collection('users')\
        .document(user_id)\
        .collection('todos').get()
```

Registrar una tarea ligada a un usuario
```python
def put_todo(user_id, description):
    todos_collection_ref = db.collection('users').document(user_id).collection('todos')
    todos_collection_ref.add({'description': description})
```    