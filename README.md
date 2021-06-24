# Elixir-Elm Stack Workshop Macquarie Google Developers Club

This is a intro to developing and testing an app in Elm and Elixir, in the future
we shall deploy it, keep tuned.

The app is a task app based on my assignments api used to track assignments.

The app will start off with a spec for the api, 
https://app.swaggerhub.com/apis-docs/BebopBamf/Tasks/1.0.0 in which we shall
build the api.

Some design decisions!

If anyone has ever used elixir before you would know the spec doesn't look
like what someone usually makes with Elixir. My background is not from Elixir
and I have chosen to conform to the normal REST api standard rather then the
elixir, where you don't have nested json e.g.

```json
{
    "tasks": {
        "name": "some name",
        "description": "some description"
    }
}
```

instead you would have it flattned

```json
{
    "name": "some name",
    "description": "some description"
}
```

and the routes aren't plural, they follow a consistant structure.

```
// elixir standard

// read all tasks
GET /tasks

// read tasks by ID
GET /tasks/{id}

// create a task
POST /tasks

// update a task
PUT /tasks/{id}
PATCH /tasks/{id}

// delete a task
DELETE /tasks/{id}

// the normal standard

// read all tasks
GET /tasks

// read tasks by ID
GET /task/{id}

// create a task
POST /task

// update a task
PUT /task/{id}
PATCH /task/{id}

// delete a task
DELETE /task/{id}
```

Another design decision I made was to return a response of different
http codes as errors, along with a json response instead of the typical html
response that you get with Phoenix, one this is to make it easier to develop the
client side. Two it made it more fun to demonstrate the POWERRR of functional
programming!

The last design decision I made was to use UNIX or POSIX timestamps instead of
the ISO standard with a timezone. This is a controversial topic, so far we have
done our best to be consistant with all the web standards, but this is not a REST
standard. The reason we do this as Elm programmers is because of a philosophy we
have of all data processing happening on the client side. It is way easier just
to store numbers, and it scales a lot better (costs less as the clients grow)
even though it's just a small thing. When you process timestamps depending on the
clients region it is considered bad practive to use Naive timestamps (without
region or timezone information). Since if the client lives in a different time
zone, then the information will be displayed wrong! As such usually you can do
two things, have a utc_timestamp which includes the utc timezone and process the
timestamp on the client side or server side to serve to the correct timestamp. Or
a much faster method is to convert the timestamp on the client side, to a format
that is relative to a fixed point in time (the unix epoch) and generate the time
stamp from that format. Which is exactly what Elm developers do!

[read more here!](https://guide.elm-lang.org/effects/time.html)

## Prerequisites

1. Preferable to have Linux (or a vagrant vm or WSL)

2. Install Postgres, Elixir and Phoenix for your given distro! 
   https://hexdocs.pm/phoenix/installation.html

3. Install NodeJS and Yarn (Or npm but the commands will be different!)

## The Elixir API

1. Create a new application with no html and no webpack with 
   `mix phx.new tasks_api --no-webpack --no-html` and choose to install
   dependencies

2. cd into the project directory `cd tasks_api`

3. edit `./config/dev.exs` with the database configuration

```elixir
config :tasks_api, TasksApi.Repo,
  username: "yourpostgresusername",
  password: "anyrandompassword",
  database: "tasks_api_dev",
  hostname: "localhost",
  show_sensitive_data_on_connection_error: true,
  pool_size: 10
```

4. create the database with `mix ecto.setup`

5. create the skeleton api with 
   `mix phx.gen.json Tasks Task task name:string description:text due_date:integer is_complete:boolean`

6. modify `lib/tasks_api/tasks/task.ex` to look like this

```elixir
defmodule TasksApi.Tasks.Task do
  use Ecto.Schema
  import Ecto.Changeset

  schema "task" do
    field :name, :string
    field :description, :string
    field :due_date, :naive_datetime
    field :is_complete, :boolean, default: false

    timestamps()
  end

  @doc false
  def changeset(task, attrs) do
    task
    |> cast(attrs, [:name, :description, :due_date, :is_complete])
    |> validate_required([:name, :description, :is_complete])
  end
end
```

notice, we remove due_date from validate_required since it is
not a required field.

7. Run migrations with `mix ecto.migrate`

8. We need to add functionality from converting from a DateTime object to a unix
   timestamp in order to conform with the API Spec, as such, add the following
   code to `lib/tasks_api/tasks.ex`

```elixir
defmodule TasksApi.Tasks do
  ...
  import Ecto.query, warn: false
  import DateTime

  ...

  @doc """
  Transforms task date to unix timestamp

  ## Examples

      iex> task_to_unix_ts(task)
      %
  """
  def task_to_unix_ts(%{due_date: nil} = task) do
    task
  end

  def task_to_unix_ts(%{due_date: due_date_param} = task) do
    %{task | due_date: to_unix(due_date_param)}
  end
end
```

Now we need to test if the code even works, we might be tempted to create a unit
test but in development that takes too long. Languages like elixir, haskell, F#
let us try out our function right away.

try testing it using the command `iex -S mix phx.server` and using
`import TasksApi.Task` or `Alias TasksApi.Task` in the repl.

8. Modify `lib/tasks_api_web/router.ex` to provide the endpoints

```elixir
  scope "/api", TasksApiWeb do
    pipe_through :api

    get "/tasks", TaskController, :index
    resources "/task", TaskController, except: [:new, :edit, :index]
  end
```

check with `mix phx.routes` that the correct endpoints are set, see the spec.

9. We need to modify the controller so that our api can receive the correct
   formatting defined by the api spec.

   e.g. `{id: 1, name: "somename", ...}` not `{task: {id: 1, name: "somename", ...}}`

```elixir
# ./lib/tasks_api_web/controllers/task_controller.ex

  def create(conn, task_params) do
    with {:ok, %Task{} = task} <- Tasks.create_task(task_params) do
      conn
      |> put_status(:created)
      |> put_resp_header("location", Routes.task_path(conn, :show, task))
      |> render("show.json", task: task)
    end
  end
  
  def update(conn, params) do
    {id, task_params} = Map.pop!(params, "id")

    task = Tasks.get_task!(id)

    with {:ok, %Task{} = task} <- Tasks.update_task(task, task_params) do
      render(conn, "show.json", task: task)
    end
  end
```

10. Now we need to modify the views so the correct json output is displayed by
   the api.

```elixir
# ./lib/tasks_api_web/views/task_view.ex

defmodule TasksApiWeb.TaskView do
  use TasksApiWeb, :view
  alias TasksApiWeb.TaskView

  def render("index.json", %{task: task}) do
    render_many(task, TaskView, "task.json")
  end

  def render("show.json", %{task: task}) do
    render_one(task, TaskView, "task.json")
  end

  def render("task.json", %{task: task}) do
    %{
      id: task.id,
      name: task.name,
      description: task.description,
      due_date: task.due_date,
      is_complete: task.is_complete
    }
  end
end
```

```elixir
# 

```


## The Elm Frontend

1. Create a new project by first running 

`mkdir tasks_site` and `cd tasks_site`

then running

`yarn init` and follow the prompts to set up the project

next we need to install some dev dependencies for the project

`yarn add -D parcel@next elm`

and add some folders to ignore to `.gitignore`

`/node_modules`

2. next we need to initialise the elm project by typing `yarn elm init` which runs the elm init
   command through yarn