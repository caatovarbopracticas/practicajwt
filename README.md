# Implementación de JWT en un API Rails

Prmero debemos crear un nuevo API, en este caso usaremos SQLite como motor de base de datos. Para esto debemos correr el siguiente comando en la terminal.

<code>rails new rails-api --api</code>

Editamos el archivo <code>Gemfile</code> para agregar las gemas necesarias, descomentar las siguientes lineas, si se encuentran en el archivo, si no, agregarlas
```
#gem 'bcrypt', '~> 3.1.7
#﻿gem 'rack-cors'
``` 

Agregamos la gema <code>knock</code> al archivo <code>Gemfile</code> que es la que usaremos para la autenticación con **JWT**.

<code>gem 'knock'﻿</code>

Instalar las nuevas gemas agregadas al <code>Gemfile</code> corriendo el siguiente comando en la terminal.

<code>bundle install</code>

Crearemos una entidad User para hacer las pruebas, en este caso usaremos **scaffold** que nos crea los métodos necesarios del controlador para el modelo User. Corremos el siguiente comando en la terminal. El último campo es necesario para poder utilizar la funcionalidad [has_secure_password](http://guides.rubyonrails.org/security.html#user-management).

<code>rails generate scaffold User name:string email:string password_digest:string</code>

Ejecutamos el siguiente comando en la terminal para realizar la migración y crear el modelo User en base de datos.

<code>rails db:migrate</code>
  

**Knock para autenticar con JWT**

Como ya incluímos la gema, solo debemos usar el generador para instalar <code>knock</code> en la aplicación, corremos el siguiente comando en la terminal.

<code>rails generate knock:install</code>

Este comando crea el archivo <code>config/initializers/knock.rb</code> en donde se pueden aplicar configuraciones avanzadas.

**Integrar knock al modelo User**

A continuación generamos el controlador con la acción necesaria para la autenticación. El generador también se encarga de agregar la ruta necesaria en el archivo <code>config/route.rb</code>.

<code>rails generate knock:token_controller user</code>

Ahora debemos incluir **has_secure_password** en nuestro modelo <code>app/models/user.rb</code> para indicar que queremos usar BCrypt para verificar contraseña con la contraseña encriptada en la base de datos. La gema bcrypt se encarga de la encriptación, el único requisito previo es incluir el atributo **password_digest**, lo cual ya hicimos al generar nuestro modelo.

**Proteger los controladores**

Primero, hagamos que nuestros controladores tengan la habilidad de ser autenticables a través de Knock editando <code>app/controllers/application_controller.rb</code>:
```
class ApplicationController < ActionController::API
	include Knock::Authenticable
end
```
Ahora, en un REST API pueden haber entidades y métodos que requieren autenticación y otros que no. Para esto editaremos el controlador de User <code>app/controllers/users_controller.rb</code>

```
#Exigimos autenticación para los métodos show y current.
before_action :authenticate_user, only: [:show, :current]
...
#Remplazamos :password_digest por :password y :password_confirmation
params.require(:user).permit(:name, :email, :password, :password_confirmation)
```
 
**Habilitar CORS**
[Cross-Origin Resource Sharing](https://www.w3.org/TR/cors/) (CORS), se refiere a la habilidad de compartir recursos de un dominio a otro. Esto puede ser vídeos, imágenes, hojas de estilo, etc. Sin embargo, debido a la [política del mismo origen](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) los navegadores no pueden realizar solicitudes AJAX desde el documento actual a otro documento pero que pertenezca a un dominio distinto. De modo que un API debe habilitar específicamente que dominios pueden acceder al API. Si es un API público entonces se puede habilitar a cualquier dominio. En nuestro caso habilitaremos a todos los dominios debido a que no queremos restricciones para probar nuestro API desde cualquier dominio, incluyendo nuestro “localhost”.

```
Rails.application.config.middleware.insert_before 0, Rack::Cors do
	allow do
		origins '*'

		resource '*',
		headers: :any,
		methods: [:get, :post, :put, :patch, :delete, :options, :head]
	end
end
```
  

**Respuesta HTTP en JSON**

Para esto ya incluimos en inicio la gema active_model_serializers. Por defecto va a incluir todos los campos de la migración (db/migrate/*_create_users.rb), pero vamos a retirar cualquier mención del atributo “password”. Primero debemos ejecutar el siguiente comando en una terminal para generar el serializer de user

<code>rails generate serializer user</code>

Luego vamos al archivo <code># app/serializers/user_serializer.rb</code>
```
class UserSerializer < ActiveModel::Serializer
	attributes :id, :name, :email
end
```

Crear el método <code>current</code> en el controlador de user <code> app/controllers/users_controller.rb</code> este método nos permitirá obtener la información del usuario que esté autenticado.

```
def current
	render json: current_user
end
```
  
Agregar las rutas al archivo <code>config/routes.rb</code>
```
Rails.application.routes.draw do  
  post ‘user_token’ => ‘user_token#create’  
  get ‘users/current’ => ‘users#current’  
  resources :users  
end
```

**Pasos Finales**
Editar el controlador <code>app/controllers/user_token_controller.rb</code>
```
class UserTokenController < Knock::AuthTokenController
	skip_before_action :verify_authenticity_token, raise: false
end
```
Modificar el archivo <code>config/initializers/knock.rb</code> agregando la siguiente linea

```
config.token_secret_signature_key = -> {Rails.application.credentials.fetch(:secret_key_base)}
```

## Pruebas

Iniciar el servidor de Ruby on Rails corriendo en la terminal el siguiente comando

<code>rails server</code>

**Crear usuarios**

Para crear los usuarios podemos ejecutar el siguiente comando desde una terminal o correrlo directamente desde POSTMAN

```
curl -H "Content-type: application/json" -d '{"user":{"name":"Luke Skywalker","email":"luke@starwars.com","password": "12345678","password_confirmation":"12345678"}}' http://localhost:3000/users
```
```
curl -H "Content-type: application/json" -d '{"user":{"name":"Han Solo","email":"han@starwars.com","password": "12345678","password_confirmation":"12345678"}}' http://localhost:3000/users
```
  
Listamos los usuarios creados

<code>http://localhost:3000/users</code>

**Autenticación**

Desde la terminal, correr el siguiente comando para autenticarse con uno de los usuarios recién creados.
```
curl -X POST http://localhost:3000/user_token -H 'Content-Type: application/json' -d '{"auth":{	"email": "luke@starwars.com", "password": "12345678"}}'
```

Si el usuario se autenticó correctamente, la respuesta que nos va a devolver el JWT:

<code>{"jwt":"este_es_el_jwt"}</code>

Obtengamos la información de nuestro propio usuario (Luke Skywalker):
```
curl -X GET http://localhost:3000/users/current -H 'Authorization: Bearer el_jwt_recibido_previamente'
 ```
Obtengamos la información de otro usuario, en este caso Han Solo:
```
curl -X GET http://localhost:3000/users/2 -H 'Authorization: Bearer el_jwt_recibido_previamente'
 ```

## Referencias
[API desarrollada con Ruby on Rails en modo API usando JWT para autenticar.](https://medium.com/@positivecarlos/api-desarrollada-con-ruby-on-rails-en-modo-api-usando-jwt-para-autenticar-26f91b71d194)
