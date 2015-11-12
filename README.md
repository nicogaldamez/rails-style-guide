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

  Para un ejemplo más claro: [RailsCast on the subject](http://railscasts.com/episodes/326-activeattr).

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
  Keep the `schema.rb` (or `structure.sql`) under version control.
<sup>[[link](#schema-version)]</sup>

* <a name="db-schema-load"></a>
  Use `rake db:schema:load` instead of `rake db:migrate` to initialize an empty
  database.
<sup>[[link](#db-schema-load)]</sup>

* <a name="default-migration-values"></a>
  Enforce default values in the migrations themselves instead of in the
  application layer.
<sup>[[link](#default-migration-values)]</sup>

  ```Ruby
  # bad - application enforced default value
  def amount
    self[:amount] or 0
  end
  ```

  While enforcing table defaults only in Rails is suggested by many
  Rails developers, it's an extremely brittle approach that
  leaves your data vulnerable to many application bugs.  And you'll
  have to consider the fact that most non-trivial apps share a
  database with other applications, so imposing data integrity from
  the Rails app is impossible.

* <a name="foreign-key-constraints"></a>
  Enforce foreign-key constraints. As of Rails 4.2, ActiveRecord
  supports foreign key constraints natively.
  <sup>[[link](#foreign-key-constraints)]</sup>

* <a name="change-vs-up-down"></a>
  When writing constructive migrations (adding tables or columns),
  use the `change` method instead of `up` and `down` methods.
  <sup>[[link](#change-vs-up-down)]</sup>

  ```Ruby
  # the old way
  class AddNameToPeople < ActiveRecord::Migration
    def up
      add_column :people, :name, :string
    end

    def down
      remove_column :people, :name
    end
  end

  # the new prefered way
  class AddNameToPeople < ActiveRecord::Migration
    def change
      add_column :people, :name, :string
    end
  end
  ```

* <a name="no-model-class-migrations"></a>
  Don't use model classes in migrations. The model classes are constantly
  evolving and at some point in the future migrations that used to work might
  stop, because of changes in the models used.
<sup>[[link](#no-model-class-migrations)]</sup>

## Views

* <a name="no-direct-model-view"></a>
  Never call the model layer directly from a view.
<sup>[[link](#no-direct-model-view)]</sup>

* <a name="no-complex-view-formatting"></a>
  Never make complex formatting in the views, export the formatting to a method
  in the view helper or the model.
<sup>[[link](#no-complex-view-formatting)]</sup>

* <a name="partials"></a>
  Mitigate code duplication by using partial templates and layouts.
<sup>[[link](#partials)]</sup>

## Internationalization

* <a name="locale-texts"></a>
  No strings or other locale specific settings should be used in the views,
  models and controllers. These texts should be moved to the locale files in the
  `config/locales` directory.
<sup>[[link](#locale-texts)]</sup>

* <a name="translated-labels"></a>
  When the labels of an ActiveRecord model need to be translated, use the
  `activerecord` scope:
<sup>[[link](#translated-labels)]</sup>

  ```
  en:
    activerecord:
      models:
        user: Member
      attributes:
        user:
          name: 'Full name'
  ```

  Then `User.model_name.human` will return "Member" and
  `User.human_attribute_name("name")` will return "Full name". These
  translations of the attributes will be used as labels in the views.


* <a name="organize-locale-files"></a>
  Separate the texts used in the views from translations of ActiveRecord
  attributes. Place the locale files for the models in a folder `locales/models` and the
  texts used in the views in folder `locales/views`.
<sup>[[link](#organize-locale-files)]</sup>

  * When organization of the locale files is done with additional directories,
    these directories must be described in the `application.rb` file in order
    to be loaded.

      ```Ruby
      # config/application.rb
      config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
      ```

* <a name="shared-localization"></a>
  Place the shared localization options, such as date or currency formats, in
  files under the root of the `locales` directory.
<sup>[[link](#shared-localization)]</sup>

* <a name="short-i18n"></a>
  Use the short form of the I18n methods: `I18n.t` instead of `I18n.translate`
  and `I18n.l` instead of `I18n.localize`.
<sup>[[link](#short-i18n)]</sup>

* <a name="lazy-lookup"></a>
  Use "lazy" lookup for the texts used in views. Let's say we have the following
  structure:
<sup>[[link](#lazy-lookup)]</sup>

  ```
  en:
    users:
      show:
        title: 'User details page'
  ```

  The value for `users.show.title` can be looked up in the template
  `app/views/users/show.html.haml` like this:

  ```Ruby
  = t '.title'
  ```

* <a name="dot-separated-keys"></a>
  Use the dot-separated keys in the controllers and models instead of specifying
  the `:scope` option. The dot-separated call is easier to read and trace the
  hierarchy.
<sup>[[link](#dot-separated-keys)]</sup>

  ```Ruby
  # bad
  I18n.t :record_invalid, scope: [:activerecord, :errors, :messages]

  # good
  I18n.t 'activerecord.errors.messages.record_invalid'
  ```

* <a name="i18n-guides"></a>
  More detailed information about the Rails I18n can be found in the [Rails
  Guides](http://guides.rubyonrails.org/i18n.html)
<sup>[[link](#i18n-guides)]</sup>

## Assets

Use the [assets pipeline](http://guides.rubyonrails.org/asset_pipeline.html) to leverage organization within
your application.

* <a name="reserve-app-assets"></a>
  Reserve `app/assets` for custom stylesheets, javascripts, or images.
<sup>[[link](#reserve-app-assets)]</sup>

* <a name="lib-assets"></a>
  Use `lib/assets` for your own libraries that don’t really fit into the
  scope of the application.
<sup>[[link](#lib-assets)]</sup>

* <a name="vendor-assets"></a>
  Third party code such as [jQuery](http://jquery.com/) or
  [bootstrap](http://twitter.github.com/bootstrap/) should be placed in
  `vendor/assets`.
<sup>[[link](#vendor-assets)]</sup>

* <a name="gem-assets"></a>
  When possible, use gemified versions of assets (e.g.
  [jquery-rails](https://github.com/rails/jquery-rails),
  [jquery-ui-rails](https://github.com/joliss/jquery-ui-rails),
  [bootstrap-sass](https://github.com/thomas-mcdonald/bootstrap-sass),
  [zurb-foundation](https://github.com/zurb/foundation)).
<sup>[[link](#gem-assets)]</sup>

## Mailers

* <a name="mailer-name"></a>
  Name the mailers `SomethingMailer`. Without the Mailer suffix it isn't
  immediately apparent what's a mailer and which views are related to the
  mailer.
<sup>[[link](#mailer-name)]</sup>

* <a name="html-plain-email"></a>
  Provide both HTML and plain-text view templates.
<sup>[[link](#html-plain-email)]</sup>

* <a name="enable-delivery-errors"></a>
  Enable errors raised on failed mail delivery in your development environment.
  The errors are disabled by default.
<sup>[[link](#enable-delivery-errors)]</sup>

  ```Ruby
  # config/environments/development.rb

  config.action_mailer.raise_delivery_errors = true
  ```

* <a name="local-smtp"></a>
  Use a local SMTP server like
  [Mailcatcher](https://github.com/sj26/mailcatcher) in the development
  environment.
<sup>[[link](#local-smtp)]</sup>

  ```Ruby
  # config/environments/development.rb

  config.action_mailer.smtp_settings = {
    address: 'localhost',
    port: 1025,
    # more settings
  }
  ```

* <a name="default-hostname"></a>
  Provide default settings for the host name.
<sup>[[link](#default-hostname)]</sup>

  ```Ruby
  # config/environments/development.rb
  config.action_mailer.default_url_options = { host: "#{local_ip}:3000" }

  # config/environments/production.rb
  config.action_mailer.default_url_options = { host: 'your_site.com' }

  # in your mailer class
  default_url_options[:host] = 'your_site.com'
  ```

* <a name="url-not-path-in-email"></a>
  If you need to use a link to your site in an email, always use the `_url`, not
  `_path` methods. The `_url` methods include the host name and the `_path`
  methods don't.
<sup>[[link](#url-not-path-in-email)]</sup>

  ```Ruby
  # bad
  You can always find more info about this course
  <%= link_to 'here', course_path(@course) %>

  # good
  You can always find more info about this course
  <%= link_to 'here', course_url(@course) %>
  ```

* <a name="email-addresses"></a>
  Format the from and to addresses properly. Use the following format:
<sup>[[link](#email-addresses)]</sup>

  ```Ruby
  # in your mailer class
  default from: 'Your Name <info@your_site.com>'
  ```

* <a name="delivery-method-test"></a>
  Make sure that the e-mail delivery method for your test environment is set to
  `test`:
<sup>[[link](#delivery-method-test)]</sup>

  ```Ruby
  # config/environments/test.rb

  config.action_mailer.delivery_method = :test
  ```

* <a name="delivery-method-smtp"></a>
  The delivery method for development and production should be `smtp`:
<sup>[[link](#delivery-method-smtp)]</sup>

  ```Ruby
  # config/environments/development.rb, config/environments/production.rb

  config.action_mailer.delivery_method = :smtp
  ```

* <a name="inline-email-styles"></a>
  When sending html emails all styles should be inline, as some mail clients
  have problems with external styles. This however makes them harder to maintain
  and leads to code duplication. There are two similar gems that transform the
  styles and put them in the corresponding html tags:
  [premailer-rails](https://github.com/fphilipe/premailer-rails) and
  [roadie](https://github.com/Mange/roadie).
<sup>[[link](#inline-email-styles)]</sup>

* <a name="background-email"></a>
  Sending emails while generating page response should be avoided. It causes
  delays in loading of the page and request can timeout if multiple email are
  sent. To overcome this emails can be sent in background process with the help
  of [sidekiq](https://github.com/mperham/sidekiq) gem.
<sup>[[link](#background-email)]</sup>

## Time

* <a name="tz-config"></a>
  Config your timezone accordingly in `application.rb`.
<sup>[[link](#tz-config)]</sup>

  ```Ruby
  config.time_zone = 'Eastern European Time'
  # optional - note it can be only :utc or :local (default is :utc)
  config.active_record.default_timezone = :local
  ```

* <a name="time-parse"></a>
  Don't use `Time.parse`.
<sup>[[link](#time-parse)]</sup>

  ```Ruby
  # bad
  Time.parse('2015-03-02 19:05:37') # => Will assume time string given is in the system's time zone.

  # good
  Time.zone.parse('2015-03-02 19:05:37') # => Mon, 02 Mar 2015 19:05:37 EET +02:00
  ```

* <a name="time-now"></a>
  Don't use `Time.now`.
<sup>[[link](#time-now)]</sup>

  ```Ruby
  # bad
  Time.now # => Returns system time and ignores your configured time zone.

  # good
  Time.zone.now # => Fri, 12 Mar 2014 22:04:47 EET +02:00
  Time.current # Same thing but shorter.
  ```

## Bundler

* <a name="dev-test-gems"></a>
  Put gems used only for development or testing in the appropriate group in the
  Gemfile.
<sup>[[link](#dev-test-gems)]</sup>

* <a name="only-good-gems"></a>
  Use only established gems in your projects. If you're contemplating on
  including some little-known gem you should do a careful review of its source
  code first.
<sup>[[link](#only-good-gems)]</sup>

* <a name="os-specific-gemfile-locks"></a>
  OS-specific gems will by default result in a constantly changing
  `Gemfile.lock` for projects with multiple developers using different operating
  systems.  Add all OS X specific gems to a `darwin` group in the Gemfile, and
  all Linux specific gems to a `linux` group:
<sup>[[link](#os-specific-gemfile-locks)]</sup>

  ```Ruby
  # Gemfile
  group :darwin do
    gem 'rb-fsevent'
    gem 'growl'
  end

  group :linux do
    gem 'rb-inotify'
  end
  ```

  To require the appropriate gems in the right environment, add the
  following to `config/application.rb`:

  ```Ruby
  platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
  Bundler.require(platform)
  ```

* <a name="gemfile-lock"></a>
  Do not remove the `Gemfile.lock` from version control. This is not some
  randomly generated file - it makes sure that all of your team members get the
  same gem versions when they do a `bundle install`.
<sup>[[link](#gemfile-lock)]</sup>

## Managing processes

* <a name="foreman"></a>
  If your projects depends on various external processes use
  [foreman](https://github.com/ddollar/foreman) to manage them.
<sup>[[link](#foreman)]</sup>

# Further Reading

There are a few excellent resources on Rails style, that you should consider if
you have time to spare:

* [The Rails 4 Way](http://www.amazon.com/The-Rails-Addison-Wesley-Professional-Ruby/dp/0321944275)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)
* [The Cucumber Book](http://pragprog.com/book/hwcuc/the-cucumber-book)
* [Everyday Rails Testing with RSpec](https://leanpub.com/everydayrailsrspec)
* [Better Specs for RSpec](http://betterspecs.org)

# Contributing

Nothing written in this guide is set in stone. It's my desire to work together
with everyone interested in Rails coding style, so that we could ultimately
create a resource that will be beneficial to the entire Ruby community.

Feel free to open tickets or send pull requests with improvements. Thanks in
advance for your help!

You can also support the project (and RuboCop) with financial contributions via
[gittip](https://www.gittip.com/bbatsov).

[![Support via Gittip](https://rawgithub.com/twolfson/gittip-badge/0.2.0/dist/gittip.png)](https://www.gittip.com/bbatsov)

## How to Contribute?

It's easy, just follow the [contribution guidelines](https://github.com/bbatsov/rails-style-guide/blob/master/CONTRIBUTING.md).

# License

![Creative Commons License](http://i.creativecommons.org/l/by/3.0/88x31.png)
This work is licensed under a [Creative Commons Attribution 3.0 Unported
License](http://creativecommons.org/licenses/by/3.0/deed.en_US)

# Spread the Word

A community-driven style guide is of little use to a community that doesn't know
about its existence. Tweet about the guide, share it with your friends and
colleagues. Every comment, suggestion or opinion we get makes the guide just a
little bit better. And we want to have the best possible guide, don't we?

Cheers,<br/>
[Bozhidar](https://twitter.com/bbatsov)
