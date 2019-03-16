---
layout: post
title:  "[reinventando a roda] Construindo google agenda"
date:   2014-05-30 20:49:51
---

Um dos melhores formas de treinar programação é recriar serviços existentes, o objetivo dessa serie vai ser recriar 
serviços existentes com proposito de aprendizado.
O serviço clonado será o google agenda. 
Vamos comecar criando nosso projeto:

{% highlight bash %}
 $ rails new pirate_agenda -d postgresql
$ bundle exec rake db:create
{% endhighlight %}

Vamos criar um simples controller para ser a pagina inicial do projeto:
{% highlight bash %}
 $ rails g controller Home index
{% endhighlight %}

isso vai nos levar a esse resultado:
<img src="{{ site.baseurl }}/assets/img/ss_first.png">

Nada impressionante, então vamos adicionar login pois nossos usuarios precisam se identificar para reservar as salas.
{% highlight bash %}
  rails generate model user name email password
rake db:migrate
{% endhighlight %}

{% highlight ruby %}
#app/models/user.rb
class User < ActiveRecord::Base
  has_secure_password
end
{% endhighlight %}

{% highlight ruby %}
#config/routes.rb
Rails.application.routes.draw do
  get 'home/index'
  root 'home#index'

  get '/signup' => 'users#new'
  post '/users' => 'users#create'
end
{% endhighlight %}

{% highlight ruby %}
#app/controllers/users_controller.rb
class UsersController < ApplicationController

    def new
      @user = User.new
    end

    def create
      @user = User.new(user_params)
      if @user.save
        session[:user_id] = @user.id
        redirect_to root_path
      else
        redirect_to '/signup'
      end
    end 

    private
    
    def user_params
      params.require(:user).permit(:name, :email, :password, :password_confirmation)
    end  

end
{% endhighlight %}

{% highlight  erb %}
<!-- app/views/users/new.html.erb -->

<h1>Signup!</h1>

<%= form_for @user do |f| %>

  Name: <%= f.text_field :name %>
  Email: <%= f.text_field :email %>
  Password: <%= f.password_field :password %>
  Password Confirmation: <%= f.password_field :password_confirmation %>
  <%= f.submit "Submit" %>

<% end %>
{% endhighlight %}

Com isso podemos criar os usuarios, vá em /signup para ver o resultado:
<img src="{{ site.baseurl }}/assets/img/ss_second.png">

a evidencia de que nosso usuario foi criado:
{% highlight  irb %}
Started POST "/users" for ::1 at 2015-06-04 22:13:53 -0300
  ActiveRecord::SchemaMigration Load (0.4ms)  SELECT "schema_migrations".* FROM "schema_migrations"
Processing by UsersController#create as HTML
  Parameters: {"utf8"=>"✓", "authenticity_token"=>"62Id5TSuYIf2ifGJlx0igZRMkgao63UAPGrEDAPgtG0HykIU1SXkiO3K9KIs112mFzwngo4bLTqTGQnWD+27RQ==", "user"=>{"name"=>"teste", "email"=>"teste@example.com", "password"=>"[FILTERED]", "password_confirmation"=>"[FILTERED]"}, "commit"=>"Submit"}
   (0.2ms)  BEGIN
  SQL (2.4ms)  INSERT INTO "users" ("name", "email", "password_digest", "created_at", "updated_at") VALUES ($1, $2, $3, $4, $5) RETURNING "id"  [["name", "teste"], ["email", "teste@example.com"], ["password_digest", "$2a$10$.v4UtvrrBxh2xFCmxYzNn.2TSd2lnilLROpXxAWxAUUBUTa5RkQMe"], ["created_at", "2015-06-05 01:13:53.787900"], ["updated_at", "2015-06-05 01:13:53.787900"]]
   (0.3ms)  COMMIT
Redirected to http://localhost:3000/
Completed 302 Found in 100ms (ActiveRecord: 5.9ms)
{% endhighlight %}


Mas nossos usuarios ainda não conseguem fazer login, vamos corrigir isso:

{% highlight  erb %}
<!-- app/views/sessions/new.html.erb -->

<h1>Login</h1>

<%= form_tag '/login' do %>

  Email: <%= text_field_tag :email %>
  Password: <%= password_field_tag :password %>
  <%= submit_tag "Submit" %>

<% end %>
{% endhighlight %}

{% highlight  ruby %}
Rails.application.routes.draw do
  get 'home/index'
  root 'home#index'

  get '/signup' => 'users#new'
  post '/users' => 'users#create'

  get '/login' => 'sessions#new'
  post '/login' => 'sessions#create'
  get '/logout' => 'sessions#destroy'
end
{% endhighlight %}

{% highlight  ruby %}
# app/controllers/sessions_controller.rb

class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by_email(params[:email])
    # If the user exists AND the password entered is correct.
    if user && user.authenticate(params[:password])
      # Save the user id inside the browser cookie. This is how we keep the user 
      # logged in when they navigate around our website.
      session[:user_id] = user.id
      redirect_to root_path
    else
    # If user's login doesn't work, send them back to the login form.
      redirect_to '/login'
    end
  end

  def destroy
    session[:user_id] = nil
    redirect_to '/login'
  end

end
{% endhighlight %}

{% highlight  ruby %}
#app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # Prevent CSRF attacks by raising an exception.
  # For APIs, you may want to use :null_session instead.
  protect_from_forgery with: :exception

  def current_user
    @current_user ||= User.find(session[:user_id]) if session[:user_id]
  end
  helper_method :current_user

  def authorize
    redirect_to login_path unless current_user
  end
end
{% endhighlight %}

{% highlight  erb %}
<!-- app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html>
<head>
  <title>PirateAgenda</title>
  <%= stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track' => true %>
  <%= javascript_include_tag 'application', 'data-turbolinks-track' => true %>
  <%= csrf_meta_tags %>
</head>
<body>

<% if current_user %>
  Signed in as <%= current_user.name %> | <%= link_to "Logout", logout_path %>
<% else %>
  <%= link_to 'Login', login_path %> | <%= link_to 'Signup', signup_path %>
<% end %>

<%= yield %>

</body>
</html>
{% endhighlight %}
Com isso podemos fazer o login:
<img src="{{ site.baseurl }}/assets/img/ss_third.png">


Agora precisamos de uma estrutura para persistir as datas e horarios dos agendamentos.
Para isso utilizarei hstore do postgres.

primeiro vamos habilitar o hstore por meio de uma migration:
{% highlight  bash %}
rails g migration enable_hstore_extension 
{% endhighlight %}

{% highlight  ruby %}
class EnableHstoreExtension < ActiveRecord::Migration
  def change
    enable_extension 'hstore'
  end
end
{% endhighlight %}

Agora podemos criar um model para armazenar as reservas com um campo hstore.
{% highlight  ruby %}
class CreateReservations < ActiveRecord::Migration
  def change
    create_table :reservations do |t|
      t.references :user, index: true, foreign_key: true
      t.hstore :position, default: {}, null: false
      t.timestamps null: false
    end
  end
end
{% endhighlight %}

{% highlight  ruby %}
#app/models/reservation.rb
class Reservation < ActiveRecord::Base
  belongs_to :user
  store_accessor :position
end
{% endhighlight %}

vamos adicionar alguns paths no routes para manipular esse model. 
{% highlight  ruby %}
resources :reservations, only: [:index, :destroy, :update]
{% endhighlight %}

Precisamos montar a agenda da semana com links para os usuários reservarem os horarios.
{% highlight  ruby %}
#app/controllers/reservations_controller.rb
class ReservationsController < ApplicationController
  before_filter :authorize
  
  def index
    @days = Date::DAYNAMES[1..5].map(&:to_sym)
    @hours = []
    now   = DateTime.now
    (6..23).each do |hour|
      @hours << DateTime.new(now.year, now.month, now.day, hour, 0, 0, 0).strftime("%H:%M")
    end
    @reservation = Reservation.last
  end

end
{% endhighlight %}

{% highlight  erb %}
<!-- app/views/reservations/index.html.erb -->
<h1>Reserva de Sala</h1>

  <div class="table-responsive">
    <table class="table">
      <thead>
        <tr>
          <th>Horário</th>
          <% @days.each do |day| %>
            <th data-day="<%= day %>"><%= day %></th>
          <% end %>
        </tr>
      </thead>
      <tbody>
        <% @hours.each do |hour| %>
          <tr>
            <td data-hour="<%= hour %>"><%= hour %></td>
            <td data-hour="<%= hour %>"><%= make_link_for_slots @reservation, :monday, hour %></td>
            <td data-hour="<%= hour %>"><%= make_link_for_slots @reservation, :tuesday, hour %></td>
            <td data-hour="<%= hour %>"><%= make_link_for_slots @reservation, :wednesday, hour %></td>
            <td data-hour="<%= hour %>"><%= make_link_for_slots @reservation, :thursday, hour %></td>
            <td data-hour="<%= hour %>"><%= make_link_for_slots @reservation, :friday, hour %></td>
          </tr>
        <% end %>
      </tbody>
    </table>
  </div>
{% endhighlight %}

Perceba que fazemos uso de um helper para facilitar a criação dos links que criam as reservas.
{% highlight  erb %}
#app/helpers/reservations_helper.rb
module ReservationsHelper
  def make_link_for_slots(slot, day, hour)
    link_to_unless  slot.position["[#{hour.to_i}, :#{day}]"], 
                    "Reservar sala", 
                    reservation_path(slot, reservation: { hour: hour, day: day.to_s}), :method => :put, remote: true, id: "#{day}-#{hour.to_i}" do
                      "Reservada" 
                    end    
  end
end
{% endhighlight %}

Como estamos reservando as salas com AJAX precisamos devolver um JS no update:
{% highlight  erb %}
$("#<%= "#{@day}-#{@hour}"%>").replaceWith("Reservada");
{% endhighlight %}

Com isso já devemos ter esse resultado, que permite ao usuario se registrar e reservar um horario na sala de 
reunião:
<img src="{{ site.baseurl }}/assets/img/ss_fourth.png">

Porém um problema da nossa abordagem é que se uma pessoa abrir o browser, sair para tomar um café 
e nesse meio tempo outra pessoa reservar o mesmo horario que ela iria reservar. Ela só perceberá depois de receber um erro.
E essa será nossa ultima tarefa, como fazer para que todos os clientes recebam a notificação de que um horario for reservado
no momento em que ele acontece?

Algumas opções vem a cabeça: 
  - polling.
  - websockets.
  - SSE(server-sent-event).

Vou usar SSE para explorar o action controller live =)


----------
links:
[How to persist hashes in Rails applications with PostgreSQL](http://goo.gl/FzxDsh)



