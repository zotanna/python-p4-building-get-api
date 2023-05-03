# Building a GET API

## Learning Goals

- Build an API to handle GET requests.

***

## Key Vocab

- **Application Programming Interface (API)**: a software application that
  allows two or more software applications to communicate with one another.
  Can be standalone or incorporated into a larger product.
- **HTTP Request Method**: assets of HTTP requests that tell the server which
  actions the client is attempting to perform on the located resource.
- **`GET`**: the most common HTTP request method. Signifies that the client is
  attempting to view the located resource.
- **`POST`**: the second most common HTTP request method. Signifies that the
  client is attempting to submit a form to create a new resource.
- **`PATCH`**: an HTTP request method that signifies that the client is attempting
  to update a resource with new information.
- **`DELETE`**: an HTTP request method that signifies that the client is
  attempting to delete a resource.

***

## Introduction

Imagine this scenario: you're given the task of creating a new game review
website from scratch. You want a dynamic, highly interactive frontend, so
naturally you choose React. You also need to store the data about your users,
your games, and the reviews somewhere. Well, it sounds like we need a database
for that. Great! We can use SQLAlchemy to set up and access data from the
database.

Here's the problem though. React can't communicate directly with the database —
for that, you need SQLAlchemy and Flask. SQLAlchemy also doesn't know
anything about your React application (and nor should it!). So then how can
we connect up our React frontend with the database?

Well, it sounds like we need some sort of **interface** between React and our
database. Perhaps some sort of **Application Programming Interface** (or as you
may know it, API). We need a structured way for these two applications to
communicate, using a couple things they **do** have in common: **HTTP** and
**JSON**.

_That_ is what we'll be building for the rest of this section: an API
(specifically, a JSON API) that will allow us to use SQLAlchemy to communicate
with a database from a React application — or really, from any application that
speaks HTTP!

***

## Setup

We'll build up our Flask application from a few models and views that are ready
to go. Run these commands to install the dependencies and set up the database:

```console
$ pipenv install; pipenv shell
$ cd server
$ flask db upgrade
$ python seed.py
```

You can view the models in the `server/models.py` module, and the migrations in the
`server/migrations/versions` directory. Here's what the relationships will look
like in our ERD:

![Game Reviews ERD](https://curriculum-content.s3.amazonaws.com/phase-3/active-record-associations-many-to-many/games-reviews-users-erd.png)

Then, run the server:

```console
$ python app.py
```

With that set up, let's work on getting Flask and SQLAlchemy working
together!

***

## Getting All Records

Imagine we're building a feature in a React application where we'd like to show
our users a list of all the games in the database. From React, we might have
code similar to the following to make this request for the data:

```jsx
function GameList() {
  const [games, setGames] = useState([]);

  useEffect(() => {
    fetch("http://localhost:9292/games")
      .then((r) => r.json())
      .then((games) => setGames(games));
  }, []);

  return (
    <section>
      {games.map((game) => (
        <GameItem key={game.id} game={game} />
      ))}
    </section>
  );
}
```

It's now our job to set up the server so that when a GET request is made to
`/games`, we return an array of all the games in our database in JSON format.
Let's set that up in Flask:

```py
# server/app.py

from flask import Flask, jsonify, make_response
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

from models import db, User, Review, Game

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///app.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.json.compact = False

migrate = Migrate(app, db)

db.init_app(app)

@app.route('/')
def index():
    return "Index for Game/Review/User API"

@app.route('/games')
def games():

    games = []
    for game in Game.query.all():
        game_dict = {
            "title": game.title,
            "genre": game.genre,
            "platform": game.platform,
            "price": game.price,
        }
        games.append(game_dict)

    response = make_response(
        jsonify(games),
        200
    )

    return response

if __name__ == '__main__':
    app.run(port=5555, debug=True)

```

We see a lot of familiar faces here, but there are a couple new elements to
explore:

- `jsonify` is a method in Flask that _serializes_ its arguments as JSON and
  returns a `Response` object. It can accept lists or dictionaries as arguments.
  Unfortunately, it will not accept models as arguments (darn!).
- `app.json.compact = False` is a configuration that has JSON responses print
  on separate lines with indentation. This adds some overhead, but if human eyes
  will be looking at your API, it's always good to have this set to `True`.
- Our query results have to be reformatted as dictionaries for `jsonify` to work
  its magic. The `__dict__` attribute cannot be used here because SQLAlchemy
  records have attributes that are nonstandard Python objects. We're leaving
  `game.id` out here because `game.title` is already set to unique.

> **NOTE: `jsonify()` is now run automatically on all dictionaries returned by
> Flask views. We'll just pass in those dictionaries from now on, but remember
> what `jsonify()`'s doing for you behind the scenes!**

Rerun `python app.py` in the console from the `server/` directory and you should
see something similar to the following:

```json
[
  {
    "genre": "Racing",
    "platform": "NES",
    "price": 29,
    "title": "Beat go these."
  },
  {
    "genre": "Puzzle",
    "platform": "XBox",
    "price": 21,
    "title": "Mission want go early appear community."
  },
  {
    "genre": "Trivia",
    "platform": "PC",
    "price": 5,
    "title": "Many choice guess prevent know."
  },
  {
    "genre": "Stealth",
    "platform": "Nintendo 3DS",
    "price": 18,
    "title": "Available once interesting page suffer middle."
  },
  {
    "genre": "Shooter",
    "platform": "DreamCast",
    "price": 35,
    "title": "Water occur choose population success."
  },
  {
    "genre": "Sandbox",
    "platform": "NES",
    "price": 42,
    "title": "Tax hear herself mean stop occur stand."
  },
  {
    "genre": "Life Simulator",
    "platform": "NES",
    "price": 48,
    "title": "Company focus particularly."
  },
  // ...
]
```

Awesome!

You also have a lot of control over how this data is returned by using
SQLAlchemy. For example, you could sort the games by title instead of the
default sort order:

```py
# example
games_by_title = Game.query.order_by(Game.title).all()
```

Or just return the first 10 games:

```py
# example
first_10_games = Game.query.limit(10).all()
```

Now that you have full control over how the server handles the response, you
have the freedom to design your API as you see fit — just think about what kind
of data you need for your frontend application.

Let's make another small adjustment to our view. By default, Flask sets
a [response header][response header] with the `Content-Type: text/html`, since
in general, web servers are used to send HTML content to browsers. Our server,
however, will be used to send JSON data, as you've seen above. We can indicate
this by changing the response header for all our routes by adding this to the
`games()` view:

```py
# server/app.py

# import, config, index

@app.route('/games')
def games():

    games = []
    for game in Game.query.all():
        game_dict = {
            "title": game.title,
            "genre": game.genre,
            "platform": game.platform,
            "price": game.price,
        }
        games.append(game_dict)

    response = make_response(
        games,
        200,
        {"Content-Type": "application/json"}
    )

    return response
```

...though this is unnecessary in this case due to `jsonify()`!

[response header]: https://developer.mozilla.org/en-US/docs/Glossary/Response_header

***

## Getting One Game Using Params

We've got our API set up to handle one feature so far: we can return a list of
all the games in the application. Let's imagine we're building another frontend
feature; this time, we want a component that will just display the details about
one specific game, including its associated reviews. Here's how that component
might look:

```jsx
function GameDetail({ gameId }) {
  const [game, setGame] = useState(null);

  useEffect(() => {
    fetch(`http://localhost:9292/games/${gameId}`)
      .then((r) => r.json())
      .then((game) => setGame(game));
  }, [gameId]);

  if (!game) return <h2>Loading game data...</h2>;

  return (
    <div>
      <h2>{game.title}</h2>
      <p>Genre: {game.genre}</p>
      <h4>Reviews</h4>
      {game.reviews.map((review) => (
        <div>
          <h5>{review.user.name}</h5>
          <p>Score: {review.score}</p>
          <p>Comment: {review.comment}</p>
        </div>
      ))}
    </div>
  );
}
```

So for this feature, we know our server needs to be able to handle a GET request
to return data about a specific game, using the game's ID to find it in the
database. For example, a `GET /games/10` request should return the game with the
ID of 10 from the database; and a `GET /games/29` request should return the game
with the ID of 29.

As we saw in the previous module, we can access data from the dynamic portion of
the URL by using the dynamic parameter name as an argument for the view. After
this, we will access the unique record using SQLAlchemy's `query.filter()`
method and returning the first result.

> **NOTE: to retrieve the one, correct result for your filter statement, you
> must use a _unique_ attribute. Make sure you only use unique attributes in
> your dynamic URLs!**

```py
# server/app.py

# import, config, index, games

@app.route('/games/<int:id>')
def game_by_id(id):
    game = Game.query.filter(Game.id == id).first()
    
    game_dict = {
        "title": game.title,
        "genre": game.genre,
        "platform": game.platform,
        "price": game.price,
    }

    response = make_response(
        game_dict,
        200
    )

    return response
```

With this code in place in the controller, try accessing the data about one game
in the browser at
[http://127.0.0.1:5555/games/1](http://127.0.0.1:5555/games/1). You should see
an object like this in the response:

```json
{
  "genre": "Racing",
  "platform": "NES",
  "price": 29,
  "title": "Beat go these."
}
```

Try making requests using other game IDs as well. As long as the ID exists in
the database, you'll get a response.

### Accessing Associated Data

Right now, our server is returning information about the game, but how can we
also access data about its associated models like the users and reviews? We
could make another endpoint for the user and review data, and make additional
requests from the frontend, but that might get messy. It would be more efficient
to return this data together along with the game data in just one single
response.

Let's take a look at the JSON being returned from the server. How does this
Python code:

```py
game = Game.query.filter(Game.id == id).first()

```

...turn into this JSON object?

```json
{
  "genre": "Racing",
  "platform": "NES",
  "price": 29,
  "title": "Beat go these."
}
```

When we're using the `jsonify()` method, Flask serializes (converts from one
format to another) the SQLAlchemy object into a JSON object by getting a list of
keys and values to pass to the client.

There are some clear limitations to this strategy. First, we need to write new
code for every attribute that we want to share with the client. This isn't very
scalable, since every new attribute requires a new line of code in `models.py`
and `app.py`. Additionally, jsonify is very fussy with even slightly complex
Python objects. (You may have already noticed this if you tried to include a
`DateTime` field earlier.)

Lucky for us, there is a better way. SQLAlchemy-serializer is configured inside
of our models, after which the model's attributes based on the column names
are easily mapped to a dictionary that we can use to create a JSON response. We
can define exactly what we want to serve to the client and how we want to serve
it.

Let's modify `models.py` to serialize the `Game` model:

```py
# server/models.py

from flask_sqlalchemy import SQLAlchemy
from sqlalchemy import MetaData
from sqlalchemy_serializer import SerializerMixin

metadata = MetaData(naming_convention={
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
})

db = SQLAlchemy(metadata=metadata)

class Game(db.Model, SerializerMixin):
    __tablename__ = 'games'

    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String, unique=True)
    genre = db.Column(db.String)
    platform = db.Column(db.String)
    price = db.Column(db.Integer)
    created_at = db.Column(db.DateTime, server_default=db.func.now())
    updated_at = db.Column(db.DateTime, onupdate=db.func.now())

    reviews = db.relationship('Review', backref='game')

    def __repr__(self):
        return f'<Game {self.title} for {self.platform}>'

# review, user

```

The simple inclusion of `SerializerMixin` adds a `to_dict()` instance method to
the `Game` model. Run `flask shell` to open up an interactive shell and
enter the following:

```console
$ game = Game.query.first()
$ game.to_dict()
# => Can not serialize type:Review
# => Can not serialize type:Review
# => Can not serialize type:Review
# => {'title': 'Beat go these.', 'created_at': '2022-09-12 16:45:47', 'id': 1, 'genre': 'Racing', 'platform': 'NES', 'reviews': [], 'price': 29, 'updated_at': None}
```

Just like that, we have a dictionary representation of our first game! There's
still a problem, though: the game's reviews can't be serialized!

In order to serialize models with related models, all models in the
relationship tree must be configured with `SerializerMixin`. Additionally,
we need to set rules to make sure that we aren't accidentally _recursively_
serializing: we don't want a game's reviews' game's reviews, after all.

Let's modify all of our models to set them up for serialization (don't worry,
it's only a couple of lines!):

```py
class Game(db.Model, SerializerMixin):
    __tablename__ = 'games'

    serialize_rules = ('-reviews.game',)

    # columns, repr

class Review(db.Model, SerializerMixin):
    __tablename__ = 'reviews'

    serialize_rules = ('-game.reviews', '-user.reviews',)
    
    # columns, repr

class User(db.Model, SerializerMixin):
    __tablename__ = 'users'

    serialize_rules = ('-reviews.user',)

    # columns, repr
```

Now rerun `flask shell` and enter the same command. You should see all of the
first game's reviews!

```console
$ game = Game.query.first()
$ game.to_dict()
# => {'price': 29, 'platform': 'NES', 'reviews': [{'updated_at': None, 'id': 157, 'user_id': 27, 'created_at': '2022-09-12 16:45:48', 'user': {'name': 'Bradley Best', 'updated_at': None, 'id': 27, 'created_at': '2022-09-12 16:45:47'}, 'comment': 'Administration against age also dinner sound single.', 'score': 2, 'game_id': 1}, {'updated_at': None, 'id': 159, 'user_id': 27, 'created_at': '2022-09-12 16:45:48', 'user': {'name': 'Bradley Best', 'updated_at': None, 'id': 27, 'created_at': '2022-09-12 16:45:47'}, 'comment': 'None minute perhaps group.', 'score': 6, 'game_id': 1}, {'updated_at': None, 'id': 518, 'user_id': 93, 'created_at': '2022-09-12 16:45:48', 'user': {'name': 'David Mills', 'updated_at': None, 'id': 93, 'created_at': '2022-09-12 16:45:47'}, 'comment': 'Story majority out store.', 'score': 0, 'game_id': 1}], 'updated_at': None, 'id': 1, 'created_at': '2022-09-12 16:45:47', 'genre': 'Racing', 'title': 'Beat go these.'}
```

Let's reconfigure the `games/<int:id>` view to show reviews with our new,
simpler strategy for serialization:

```py
# server/app.py
@app.route('/games/<int:id>')
def game_by_id(id):
    game = Game.query.filter(Game.id == id).first()
    
    game_dict = game.to_dict()

    response = make_response(
        # it still needs to be JSON, after all
        jsonify(game_dict),
        200
    )
    response.headers["Content-Type"] = "application/json"

    return response

```

Run your server again with `flask run` and navigate to
[http://127.0.0.1:5555/games/1](http://127.0.0.1:5555/games/1) to see your
fully fleshed-out `GET` API:

```json
{
  "created_at": "2022-09-12 16:45:47",
  "genre": "Racing",
  "id": 1,
  "platform": "NES",
  "price": 29,
  "reviews": [
    {
      "comment": "Administration against age also dinner sound single.",
      "created_at": "2022-09-12 16:45:48",
      "game_id": 1,
      "id": 157,
      "score": 2,
      "updated_at": null,
      "user": {
        "created_at": "2022-09-12 16:45:47",
        "id": 27,
        "name": "Bradley Best",
        "updated_at": null
      },
      "user_id": 27
    },
    {
      "comment": "None minute perhaps group.",
      "created_at": "2022-09-12 16:45:48",
      "game_id": 1,
      "id": 159,
      "score": 6,
      "updated_at": null,
      "user": {
        "created_at": "2022-09-12 16:45:47",
        "id": 27,
        "name": "Bradley Best",
        "updated_at": null
      },
      "user_id": 27
    },
    {
      "comment": "Story majority out store.",
      "created_at": "2022-09-12 16:45:48",
      "game_id": 1,
      "id": 518,
      "score": 0,
      "updated_at": null,
      "user": {
        "created_at": "2022-09-12 16:45:47",
        "id": 93,
        "name": "David Mills",
        "updated_at": null
      },
      "user_id": 93
    }
  ],
  "title": "Beat go these.",
  "updated_at": null
}
```

***

## Conclusion

In this lesson, you created your very first web API! You learned how to set up
multiple routes to handle different requests based on what kind of data we
needed for a frontend application, and used `jsonify()` and
`SQLAlchemy-serializer` to serialize the JSON response to include all the data
needed. At their most basic levels, almost all web APIs provide a way for
clients, like React applications, to interact with a database and gain access
to data in a structured way. Thanks to tools like Flask and SQLAlchemy, setting
up this interface is fairly straightforward.

***

## Solution Code

```py
# server/app.py

#!/usr/bin/env python3

from flask import Flask, jsonify, make_response
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

from models import db, User, Review, Game

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///app.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.json.compact = False

migrate = Migrate(app, db)

db.init_app(app)

@app.route('/')
def index():
    return "Index for Game/Review/User API"

@app.route('/games')
def games():

    games = []
    for game in Game.query.all():
        game_dict = {
            "title": game.title,
            "genre": game.genre,
            "platform": game.platform,
            "price": game.price,
        }
        games.append(game_dict)

    response = make_response(
        games,
        200
    )

    return response

@app.route('/games/<int:id>')
def game_by_id(id):
    game = Game.query.filter(Game.id == id).first()
    
    game_dict = game.to_dict()

    response = make_response(
        game_dict,
        200
    )

    return response

if __name__ == '__main__':
    app.run(port=5555, debug=True)

```

```py
# server/models.py

from flask_sqlalchemy import SQLAlchemy
from sqlalchemy import MetaData
from sqlalchemy_serializer import SerializerMixin

metadata = MetaData(naming_convention={
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
})

db = SQLAlchemy(metadata=metadata)

class Game(db.Model, SerializerMixin):
    __tablename__ = 'games'

    serialize_rules = ('-reviews.game',)

    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String, unique=True)
    genre = db.Column(db.String)
    platform = db.Column(db.String)
    price = db.Column(db.Integer)
    created_at = db.Column(db.DateTime, server_default=db.func.now())
    updated_at = db.Column(db.DateTime, onupdate=db.func.now())

    reviews = db.relationship('Review', backref='game')

    def __repr__(self):
        return f'<Game {self.title} for {self.platform}>'

class Review(db.Model, SerializerMixin):
    __tablename__ = 'reviews'

    serialize_rules = ('-game.reviews', '-user.reviews',)
    
    id = db.Column(db.Integer, primary_key=True)
    score = db.Column(db.Integer)
    comment = db.Column(db.String)
    created_at = db.Column(db.DateTime, server_default=db.func.now())
    updated_at = db.Column(db.DateTime, onupdate=db.func.now())

    game_id = db.Column(db.Integer, db.ForeignKey('games.id'))
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'))

    def __repr__(self):
        return f'<Review ({self.id}) of {self.game}: {self.score}/10>'

class User(db.Model, SerializerMixin):
    __tablename__ = 'users'

    serialize_rules = ('-reviews.user',)

    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String)
    
    created_at = db.Column(db.DateTime, server_default=db.func.now())
    updated_at = db.Column(db.DateTime, onupdate=db.func.now())

    reviews = db.relationship('Review', backref='user')

```

***

## Resources

- [Flask - Pallets](https://flask.palletsprojects.com/en/2.2.x/)
- [GET - Mozilla](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET)
- [flask.json.jsonify Example Code - Full Stack Python](https://www.fullstackpython.com/flask-json-jsonify-examples.html)
- [SQLAlchemy-serializer - PyPI](https://pypi.org/project/SQLAlchemy-serializer/)
