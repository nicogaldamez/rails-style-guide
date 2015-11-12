# Objetivo

El objetivo de esta guía consiste en presentar un conjunto de buenas prácticas para el desarrollo de aplicaciones en Ruby on Rails 4. La misma está basada en [Ruby coding style guide](https://github.com/bbatsov/rails-style-guide) de [bbatsov](https://github.com/bbatsov).

## Tabla de Contenidos

* [Configuración](#configuration)
* [Routing](#routing)
* [Controllers](#controllers)
* [Models](#models)
  * [ActiveRecord](#activerecord)
  * [ActiveRecord Queries](#activerecord-queries)
* [Migrations](#migrations)
* [Views](#views)
* [Internationalization](#internationalization)
* [Assets](#assets)
* [Mailers](#mailers)
* [Time](#time)
* [Bundler](#bundler)
* [Managing processes](#managing-processes)

## Configuration

* <a name="config-initializers"></a>
  Las configuraciones de inicialización deben ir en `config/initializers`. El código ubicado en initializers es ejecutado al inicio de la aplicación.
<sup>[[link](#config-initializers)]</sup>

* <a name="gem-initializers"></a>
  Mantener la inicialización de cada gema en un archivo separado con el mismo nombre de la gema, por ejemplo `carrierwave.rb`, `active_admin.rb`, etc.
<sup>[[link](#gem-initializers)]</sup>

* <a name="dev-test-prod-configs"></a>
  Los ajustes particulares a cada entorno (development, test y production) deben ir en el archivo correspondiente (`config/environments/`)
<sup>[[link](#dev-test-prod-configs)]</sup>

  * Marcar assets adicionales para precompilación (en caso de ser necesario):

    ```Ruby
    # config/environments/production.rb
    # Precompile additional assets (application.js, application.css,
    #and all non-JS/CSS are already added)
    config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
    ```

* <a name="app-config"></a>
  Mantener la configuración común a todos los entornos en el archivo `config/application.rb`.
<sup>[[link](#app-config)]</sup>

* <a name="staging-like-prod"></a>
  Crear un entorno adicional `staging` "similar" al entorno `production`.
<sup>[[link](#staging-like-prod)]</sup>

* <a name="yaml-config"></a>
  Mantener cualquier configuración adicional en archivos YAML en el directorio `config/`.
<sup>[[link](#yaml-config)]</sup>

  A partir de Rails 4.2 los archivos de configuración YAML puede ser facilmente cargados con el método `config_for`:

  ```Ruby
  Rails::Application.config_for(:yaml_file)
  ```

## Routing

* <a name="member-collection-routes"></a>
 Cuando se necesitan añadir acciones a un resource RESTful se deben usar las rutas `member` y `collection`.
<sup>[[link](#member-collection-routes)]</sup>

  ```Ruby
  # :(
  get 'subscriptions/:id/unsubscribe'
  resources :subscriptions

  # :)
  resources :subscriptions do
    get 'unsubscribe', on: :member
  end

  # :(
  get 'photos/search'
  resources :photos

  # :)
  resources :photos do
    get 'search', on: :collection
  end
  ```

* <a name="many-member-collection-routes"></a>
  Si se necesitan definir múltiples rutas `member/collection` se debe utilizar la sintaxis de bloque.
<sup>[[link](#many-member-collection-routes)]</sup>

  ```Ruby
  resources :subscriptions do
    member do
      get 'unsubscribe'
      # más rutas
    end
  end

  resources :photos do
    collection do
      get 'search'
      # más rutas
    end
  end
  ```

* <a name="nested-routes"></a>
  Utilizar nested routes para expresar de una mejor forma la relación entre modelos ActiveRecord.
<sup>[[link](#nested-routes)]</sup>

  ```Ruby
  class Post < ActiveRecord::Base
    has_many :comments
  end

  class Comments < ActiveRecord::Base
    belongs_to :post
  end

  # routes.rb
  resources :posts do
    resources :comments
  end
  ```
  
* <a name="namespaced-routes"></a>
  Si se necesitan rutas nested de más de un nivel se debe utilizar la opción `shallow: true` option. De esta forma se evitan largas urls como `posts/1/comments/5/versions/7/edit` y largos url helpers como `edit_post_comment_version`.
  
  ```Ruby
  resources :posts, shallow: true do
    resources :comments do
      resources :versions
    end
  end
  ```

* <a name="namespaced-routes"></a>
  Utilizar namespaced routes para agrupar acciones relacionadas.
<sup>[[link](#namespaced-routes)]</sup>

  ```Ruby
  namespace :admin do
    # Direcciona /admin/products/* a Admin::ProductsController
    # (app/controllers/admin/products_controller.rb)
    resources :products
  end
  ```

* <a name="no-wild-routes"></a>
 Nunca utilizar rutas de esta forma ya que habilitan el acceso a todas las rutas del controller a través de requests GET.
<sup>[[link](#no-wild-routes)]</sup>

  ```Ruby
  # :(
  match ':controller(/:action(/:id(.:format)))'
  ```

* <a name="no-match-routes"></a>
  No utilizar `match` para refinir rutas a no ser que sea necesario mappear múltiples tipos de requests `[:get, :post, :patch, :put, :delete]` a una única acción. Utilizar la opción `:via`.
<sup>[[link](#no-match-routes)]</sup>

## Controllers

* <a name="skinny-controllers"></a>
  Mantener los controllers "livianos" - sólo deben recuperar datos para la vista y no deberían contener lógica de negocio (la lógica de negocio está contenida en los models).
<sup>[[link](#skinny-controllers)]</sup>

* <a name="one-method"></a>
  Cada acción del controller debe (idealmente) invocar un solo método además de un find o new.
<sup>[[link](#one-method)]</sup>

* <a name="shared-instance-variables"></a>
  Compartir no más de 2 variables de instancias entre controller y view.
<sup>[[link](#shared-instance-variables)]</sup>

## Models

* <a name="meaningful-model-names"></a>
  Nombrar los modelos con nombre significativos (pero cortos) sin abreviaciones.
<sup>[[link](#meaningful-model-names)]</sup>

* <a name="activeattr-gem"></a>
  Si se necesitan modelos que soporten comportamiento de ActiveRecord (como validaciones) sin funcionalidad de base datos utilizar la gema [ActiveAttr](https://github.com/cgriego/active_attr).
<sup>[[link](#activeattr-gem)]</sup>

  ```Ruby
  class Message
    include ActiveAttr::Model

    attribute :name
    attribute :email
    attribute :content
    attribute :priority

    attr_accessible :name, :email, :content

    validates :name, presence: true
    validates :email, format: { with: /\A[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4}\z/i }
    validates :content, length: { maximum: 500 }
  end
  ```

  Para un ejemplo más claro: [RailsCast](http://railscasts.com/episodes/326-activeattr).

### ActiveRecord

* <a name="keep-ar-defaults"></a>
  Apegarse a la configuración por defecto de ActiveRecord (nombre de talbas, primary key, etc) a no ser que se tenga una buena razón para no hacerlo.
<sup>[[link](#keep-ar-defaults)]</sup>

  ```Ruby
  # :(
  class Transaction < ActiveRecord::Base
    self.table_name = 'order'
    ...
  end
  ```

* <a name="macro-style-methods"></a>
  Agrupar métodos como `has_many`, `validates`, etc al comienzo de la definición de la clase.
<sup>[[link](#macro-style-methods)]</sup>

  ```Ruby
  class User < ActiveRecord::Base
    # mantener el default_scope al comienzo
    default_scope { where(active: true) }

    # luego las constantes
    COLORS = %w(red green blue)

    # luego macros attr
    attr_accessor :formatted_date_of_birth

    attr_accessible :login, :first_name, :last_name, :email, :password

    # asociaciones
    belongs_to :country

    has_many :authentications, dependent: :destroy

    # validaciones
    validates :email, presence: true
    validates :username, presence: true
    validates :username, uniqueness: { case_sensitive: false }
    validates :username, format: { with: /\A[A-Za-z][A-Za-z0-9._-]{2,19}\z/ }
    validates :password, format: { with: /\A\S{8,128}\z/, allow_nil: true }

    # callbacks
    before_save :cook
    before_save :update_username_lower

    # otras macros (como devise) deben ir después de los callbacks

    ...
  end
  ```

* <a name="has-many-through"></a>
  Escoger `has_many :through` ante `has_and_belongs_to_many`. La utilización de `has_many
  :through` permite añadir atributos adiciones y validaciones en el modelo intermedio.
<sup>[[link](#has-many-through)]</sup>

  ```Ruby
  # :/
  class User < ActiveRecord::Base
    has_and_belongs_to_many :groups
  end

  class Group < ActiveRecord::Base
    has_and_belongs_to_many :users
  end

  # :)
  class User < ActiveRecord::Base
    has_many :memberships
    has_many :groups, through: :memberships
  end

  class Membership < ActiveRecord::Base
    belongs_to :user
    belongs_to :group
  end

  class Group < ActiveRecord::Base
    has_many :memberships
    has_many :users, through: :memberships
  end
  ```

* <a name="read-attribute"></a>
  Escoger `self[:attribute]` ante `read_attribute(:attribute)`.
<sup>[[link](#read-attribute)]</sup>

  ```Ruby
  # :(
  def amount
    read_attribute(:amount) * 100
  end

  # :)
  def amount
    self[:amount] * 100
  end
  ```

* <a name="write-attribute"></a>
  Escoger `self[:attribute] = value` ante `write_attribute(:attribute, value)`.
<sup>[[link](#write-attribute)]</sup>

  ```Ruby
  # :(
  def amount
    write_attribute(:amount, 100)
  end

  # :)
  def amount
    self[:amount] = 100
  end
  ```

* <a name="sexy-validations"></a>
  Utilizar las ["sexy"
  validations](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/).
<sup>[[link](#sexy-validations)]</sup>

  ```Ruby
  # :(
  validates_presence_of :email
  validates_length_of :email, maximum: 100

  # :)
  validates :email, presence: true, length: { maximum: 100 }
  ```

* <a name="custom-validator-file"></a>
  Cuando se utiliza una custom validation más de una vez o la validación es un mapeo de expresión regular se debe crear un custom validator.
<sup>[[link](#custom-validator-file)]</sup>

  ```Ruby
  # :(
  class Person
    validates :email, format: { with: /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i }
  end

  # :)
  class EmailValidator < ActiveModel::EachValidator
    def validate_each(record, attribute, value)
      record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
    end
  end

  class Person
    validates :email, email: true
  end
  ```

* <a name="app-validators"></a>
  Mantener los custom validators en `app/validators`.
<sup>[[link](#app-validators)]</sup>

* <a name="custom-validators-gem"></a>
  Considerar extrar los custom validators en una gema compartida si se va a utilizar en múltiples projectos.
<sup>[[link](#custom-validators-gem)]</sup>

* <a name="named-scopes"></a>
  Utilizar todos los scopes que se desee.
<sup>[[link](#named-scopes)]</sup>

  ```Ruby
  class User < ActiveRecord::Base
    scope :active, -> { where(active: true) }
    scope :inactive, -> { where(active: false) }

    scope :with_orders, -> { joins(:orders).select('distinct(users.id)') }
  end
  ```

* <a name="named-scope-class"></a>
  Cuando un scope definido con lambra y los parámetros se tornan muy complejos es preferible crear un método de clase y retornar un objeto `ActiveRecord::Relation`. 

<sup>[[link](#named-scope-class)]</sup>

  ```Ruby
  class User < ActiveRecord::Base
    def self.with_orders
      joins(:orders).select('distinct(users.id)')
    end
  end
  ```

* <a name="beware-update-attribute"></a>
  Tener en cuenta el comportamiento del método
  [`update_attribute`](http://api.rubyonrails.org/classes/ActiveRecord/Persistence.html#method-i-update_attribute)
  . No ejecuta las validaciones del modelo.
<sup>[[link](#beware-update-attribute)]</sup>

* <a name="user-friendly-urls"></a>
  Utilizar friendly URLs. Mostrar un atributo descriptivo del modelo en la URL y el `id`.  Hay más de una forma de realizar esto:
<sup>[[link](#user-friendly-urls)]</sup>

  * Sobreescribir el método `to_param` del modelo. Este método es utilizado por Rails para construir la URL para el objeto. La implementación por default retorna el `id` del registro como String. Puede ser sobreescrito para incluir otro atributo más amigable.

      ```Ruby
      class Person
        def to_param
          "#{id} #{name}".parameterize
        end
      end
      ```

  Para convertir la URL, debe ser invocado `parameterize` en el string. El atributo `id` del objeto debe estar al compienzo para que luego pueda ser encontrado por el método `find` de ActiveRecord.

  * Utilizar la gema `friendly_id` gem. Permite crear URLs amigables utilizando algún atributo descriptivo del modelo en lugar del `id`.

      ```Ruby
      class Person
        extend FriendlyId
        friendly_id :name, use: :slugged
      end
      ```

  Ver la [gema](https://github.com/norman/friendly_id) para obtener más información.

* <a name="find-each"></a>
  Utilizar `find_each` para iterar sobre objetos AR. Iterar sobre una colección de registros de una Base de Datos (utilizando el método `all`, por ejemplo) es muy ineficiente ya que intentará instanciar todos los objetos al mismo tiempo.
  En ese caso, se deberían utilizar métodos de procesamiento en lotes para reducir el consumo de memoria.
<sup>[[link](#find-each)]</sup>


  ```Ruby
  # :(
  Person.all.each do |person|
    person.do_awesome_stuff
  end

  Person.where('age > 21').each do |person|
    person.party_all_night!
  end

  # :)
  Person.find_each do |person|
    person.do_awesome_stuff
  end

  Person.where('age > 21').find_each do |person|
    person.party_all_night!
  end
  ```

* <a name="before_destroy"></a>
  Dado que [Rails crea callbacks para asociaciones dependientes](https://github.com/rails/rails/issues/3458), siempre invoca al callback `before_destroy` que realiza validaciones con `prepend: true`.
<sup>[[link](#before_destroy)]</sup>

  ```Ruby
  # :( (los roles serán eliminados automaticamente a pesar de ser super_admin? es true)
  has_many :roles, dependent: :destroy

  before_destroy :ensure_deletable

  def ensure_deletable
    fail "Cannot delete super admin." if super_admin?
  end

  # :)
  has_many :roles, dependent: :destroy

  before_destroy :ensure_deletable, prepend: true

  def ensure_deletable
    fail "No se puede eliminar el administrador." if super_admin?
  end
  ```

### ActiveRecord Queries

* <a name="avoid-interpolation"></a>
  Evitar la interpolación en las queries, previniendo ataques por SQL injection.
<sup>[[link](#avoid-interpolation)]</sup>

  ```Ruby
  # :( 
  Client.where("orders_count = #{params[:orders]}")

  # :)
  Client.where('orders_count = ?', params[:orders])
  ```

* <a name="named-placeholder"></a>
  Considerar la utilización de placeholders nombrados en lugar de placeholders posicionales.
<sup>[[link](#named-placeholder)]</sup>

  ```Ruby
  # :/
  Client.where(
    'created_at >= ? AND created_at <= ?',
    params[:start_date], params[:end_date]
  )

  # :)
  Client.where(
    'created_at >= :start_date AND created_at <= :end_date',
    start_date: params[:start_date], end_date: params[:end_date]
  )
  ```

* <a name="find"></a>
  Utilizar `find` en lugar de `where` cuando se necesita recuperar un único registro por id.
<sup>[[link](#find)]</sup>

  ```Ruby
  # :(
  User.where(id: id).take

  # :)
  User.find(id)
  ```

* <a name="find_by"></a>
  Utilizar `find_by` en lugar de `where` cuando se necesita recuperar un único registro por varios atributos.
<sup>[[link](#find_by)]</sup>

  ```Ruby
  # :(
  User.where(first_name: 'Bruce', last_name: 'Wayne').first

  # :)
  User.find_by(first_name: 'Bruce', last_name: 'Wayne')
  ```

* <a name="find_each"></a>
  Utilizar `find_each` cuando se necesite procesar muchos registros.
<sup>[[link](#find_each)]</sup>

  ```Ruby
  # :(
  User.all.each do |user|
    NewsMailer.weekly(user).deliver_now
  end

  # :)
  User.find_each do |user|
    NewsMailer.weekly(user).deliver_now
  end
  ```

* <a name="where-not"></a>
  Utilizar `where.not` en lugar de SQL.
<sup>[[link](#where-not)]</sup>

  ```Ruby
  # :(
  User.where("id != ?", id)

  # :)
  User.where.not(id: id)
  ```
* <a name="squished-heredocs"></a>
  Cuando se especifica una query en un método como `find_by_sql`, utilizar `squish`. Este permite formatear el SQL de manera legible a través de saltos de línea e indentaciones.
<sup>[[link](#squished-heredocs)]</sup>

  ```Ruby
  User.find_by_sql(<<SQL.squish)
    SELECT
      users.id, accounts.plan
    FROM
      users
    INNER JOIN
      accounts
    ON
      accounts.user_id = users.id
    # further complexities...
  SQL
  ```

## Migrations

* <a name="schema-version"></a>
  Mantener `schema.rb` (o `structure.sql`) bajo control de versiones.
<sup>[[link](#schema-version)]</sup>

* <a name="db-schema-load"></a>
  Utilizar `rake db:schema:load` en lugar de `rake db:migrate` para inicializar una base de datos vacía.
<sup>[[link](#db-schema-load)]</sup>

* <a name="default-migration-values"></a>
  Utilizar valores por default en las migraciones en lugar de ponerlos en la capa de aplicación.
<sup>[[link](#default-migration-values)]</sup>

  ```Ruby
  # :(
  def amount
    self[:amount] or 0
  end
  ```

* <a name="foreign-key-constraints"></a>
  Utilizar restricciones de foreign-key. 
  <sup>[[link](#foreign-key-constraints)]</sup>

* <a name="change-vs-up-down"></a>
  Cuando se escriben migraciones que añaden tablas o columnas, utilizar el método `change` en lugar de los métodos `up` y `down`.
  <sup>[[link](#change-vs-up-down)]</sup>

  ```Ruby
  # :(
  class AddNameToPeople < ActiveRecord::Migration
    def up
      add_column :people, :name, :string
    end

    def down
      remove_column :people, :name
    end
  end

  # :)
  class AddNameToPeople < ActiveRecord::Migration
    def change
      add_column :people, :name, :string
    end
  end
  ```

* <a name="no-model-class-migrations"></a>
  No utilizar métodos de clase de los modelos en las migraciones por si el modelo cambia.
<sup>[[link](#no-model-class-migrations)]</sup>

## Views

* <a name="no-direct-model-view"></a>
  Nunca llamar al modelo en las vistas.
<sup>[[link](#no-direct-model-view)]</sup>

* <a name="no-complex-view-formatting"></a>
  No utilizar formateo complejo en las vistas, exportar el formateo a un método en un helpero o modelo.
<sup>[[link](#no-complex-view-formatting)]</sup>

* <a name="partials"></a>
  Mitigar código duplicado utilizando partial y layouts.
<sup>[[link](#partials)]</sup>

## Internationalization

* <a name="locale-texts"></a>
  Ningún string u otra configuración específica de localización debe ir en las vistas, modelos o controllers. Estos textos deben estar en los archivos ubicados en el directorio `config/locales`.
<sup>[[link](#locale-texts)]</sup>

* <a name="translated-labels"></a>
  Cuando los labels de un modelo ActiveRecord deben ser traducidos se debe utilizar el scope `activerecord`:
<sup>[[link](#translated-labels)]</sup>

  ```
  en:
    activerecord:
      models:
        user: Miembro
      attributes:
        user:
          name: 'Nombre Completo'
  ```

  Luego `User.model_name.human` retornará "Miembto" y
  `User.human_attribute_name("name")` retornará "Nombre Completo". Estas traducciones serán utilizadas como labels en las vistas.


* <a name="organize-locale-files"></a>
  Separar los textos de las vistas de las traducciones de los atributos ActiveRecord. Ubicar los archivos de localización en el directorio `locales/models` y los textos utilizados en las vistas en el directorio `locales/views`.
<sup>[[link](#organize-locale-files)]</sup>

  * Cuando la organización de los archivos de localización se realiza en múltiples directorios, estos directorios deben ser configurados en el archivo `application.rb` para que sean incluídos.

      ```Ruby
      # config/application.rb
      config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
      ```

* <a name="shared-localization"></a>
  Ubicar las opciones de localización generales, como fechas, moneda, en los archivos en la raíz del directorio `locales`.
<sup>[[link](#shared-localization)]</sup>

* <a name="short-i18n"></a>
  Utilizar la forma corta de los métodos I18n: `I18n.t` en lugar de `I18n.translate`
  y `I18n.l` en lugar de `I18n.localize`.
<sup>[[link](#short-i18n)]</sup>

* <a name="lazy-lookup"></a>
  Utilizar "lazy" lookup para los textos de las vistas. Por ej, si tenemos:
<sup>[[link](#lazy-lookup)]</sup>

  ```
  en:
    users:
      show:
        title: 'Detalles del usuario'
  ```

  El valor para `users.show.title` puede ser utilizado en `app/views/users/show.html.haml` de esta forma:

  ```Ruby
  = t '.title'
  ```

* <a name="dot-separated-keys"></a>
  Utilizar la sintaxis de punto en controllers y models en lugar de especificar la opción `:scope`.
<sup>[[link](#dot-separated-keys)]</sup>

  ```Ruby
  # :(
  I18n.t :record_invalid, scope: [:activerecord, :errors, :messages]

  # :)
  I18n.t 'activerecord.errors.messages.record_invalid'
  ```

* <a name="i18n-guides"></a>
  Más información de Rails I18n: [Rails
  Guides](http://guides.rubyonrails.org/i18n.html)
<sup>[[link](#i18n-guides)]</sup>

## Assets

Utilziar el [assets pipeline](http://guides.rubyonrails.org/asset_pipeline.html) .

* <a name="reserve-app-assets"></a>
  Reservar `app/assets` para css, javascripts, e images personalizadas.
<sup>[[link](#reserve-app-assets)]</sup>

* <a name="lib-assets"></a>
  Utilizar `lib/assets` para tus propias bibliotecas que no son específicas del proyecto.
<sup>[[link](#lib-assets)]</sup>

* <a name="vendor-assets"></a>
  Código de terceros como [jQuery](http://jquery.com/) or
  [bootstrap](http://twitter.github.com/bootstrap/) debe ubicarse en `vendor/assets`.
<sup>[[link](#vendor-assets)]</sup>

## Mailers

* <a name="mailer-name"></a>
  Nombrar los mailer así: `SomethingMailer`. .
<sup>[[link](#mailer-name)]</sup>

* <a name="html-plain-email"></a>
  Proveer las versiones HTML y texto plano.
<sup>[[link](#html-plain-email)]</sup>

* <a name="enable-delivery-errors"></a>
  Habilitar los errores en el entorno de desarrollo. Estos errores son deshabilitados por defecto
<sup>[[link](#enable-delivery-errors)]</sup>

  ```Ruby
  # config/environments/development.rb

  config.action_mailer.raise_delivery_errors = true
  ```
* <a name="default-hostname"></a>
  Proveer una configuración por defecto para el host name.
<sup>[[link](#default-hostname)]</sup>

  ```Ruby
  # config/environments/development.rb
  config.action_mailer.default_url_options = { host: "#{local_ip}:3000" }

  # config/environments/production.rb
  config.action_mailer.default_url_options = { host: 'your_site.com' }

  # en la clase mailer
  default_url_options[:host] = 'your_site.com'
  ```

* <a name="url-not-path-in-email"></a>
  Si se necesitan utilizar links en el email, siempre utilizar `_url`, en lugar de
  `_path`. `_url` incluye el nombre del host.
<sup>[[link](#url-not-path-in-email)]</sup>

  ```Ruby
  # :(
  Encuentra más información sobre el curso
  <%= link_to 'here', course_path(@course) %>

  # :)
  Encuentra más información sobre el curso
  <%= link_to 'here', course_url(@course) %>
  ```

* <a name="email-addresses"></a>
  Setear correctamente from y to. Utilizar el siguiente formato:
<sup>[[link](#email-addresses)]</sup>

  ```Ruby
  # en la clase mailer
  default from: 'Tu Nombre <info@your_site.com>'
  ```

* <a name="delivery-method-test"></a>
  Asegurarse que el método delivery en test está seteado como `test`:
<sup>[[link](#delivery-method-test)]</sup>

  ```Ruby
  # config/environments/test.rb

  config.action_mailer.delivery_method = :test
  ```

* <a name="delivery-method-smtp"></a>
  El método de delivery en development y production debe ser `smtp`:
<sup>[[link](#delivery-method-smtp)]</sup>

  ```Ruby
  # config/environments/development.rb, config/environments/production.rb

  config.action_mailer.delivery_method = :smtp
  ```

* <a name="inline-email-styles"></a>
  Cuando se envían emails los estilos deben estar inline. 2 gemas para facilitar esto:
  [premailer-rails](https://github.com/fphilipe/premailer-rails) and
  [roadie](https://github.com/Mange/roadie).
<sup>[[link](#inline-email-styles)]</sup>

* <a name="background-email"></a>
  Enviar emails mientras se genera la respuesta debe evitarse. Para ello utilizar la gema [sidekiq](https://github.com/mperham/sidekiq).
<sup>[[link](#background-email)]</sup>

## Time

* <a name="tz-config"></a>
  Configurar el timezone en `application.rb`.
<sup>[[link](#tz-config)]</sup>

  ```Ruby
  config.time_zone = 'Eastern European Time'
  # optional - note it can be only :utc or :local (default is :utc)
  config.active_record.default_timezone = :local
  ```

* <a name="time-parse"></a>
  No utilizar `Time.parse`.
<sup>[[link](#time-parse)]</sup>

  ```Ruby
  # :(
  Time.parse('2015-03-02 19:05:37') # => Will assume time string given is in the system's time zone.

  # :)
  Time.zone.parse('2015-03-02 19:05:37') # => Mon, 02 Mar 2015 19:05:37 EET +02:00
  ```

* <a name="time-now"></a>
  No utilizar `Time.now`.
<sup>[[link](#time-now)]</sup>

  ```Ruby
  # :(
  Time.now # => Returns system time and ignores your configured time zone.

  # :)
  Time.zone.now # => Fri, 12 Mar 2014 22:04:47 EET +02:00
  Time.current # Same thing but shorter.
  ```

## Bundler

* <a name="dev-test-gems"></a>
  Ubicar las gemas utilizar solo para desarrollo o testing en el grupo apropiado en el Gemfile.
<sup>[[link](#dev-test-gems)]</sup>

* <a name="os-specific-gemfile-locks"></a>
  Las gemas específicas de un sistema operativo deben ubicarse en el grupo correspondiente:
<sup>[[link](#os-specific-gemfile-locks)]</sup>

  ```Ruby
  # Gemfile
  group :darwin do # OSX
    gem 'rb-fsevent'
    gem 'growl'
  end

  group :linux do
    gem 'rb-inotify'
  end
  ```

  Para requerir las gemas apropiadas según el entorno añadir lo siguiente a `config/application.rb`:

  ```Ruby
  platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
  Bundler.require(platform)
  ```

* <a name="gemfile-lock"></a>
  No quitar `Gemfile.lock` del control de versiones.
<sup>[[link](#gemfile-lock)]</sup>


# Más Info

* [The Rails 4 Way](http://www.amazon.com/The-Rails-Addison-Wesley-Professional-Ruby/dp/0321944275)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)
* [The Cucumber Book](http://pragprog.com/book/hwcuc/the-cucumber-book)
* [Everyday Rails Testing with RSpec](https://leanpub.com/everydayrailsrspec)
* [Better Specs for RSpec](http://betterspecs.org)

# License

![Creative Commons License](http://i.creativecommons.org/l/by/3.0/88x31.png)
This work is licensed under a [Creative Commons Attribution 3.0 Unported
License](http://creativecommons.org/licenses/by/3.0/deed.en_US)

