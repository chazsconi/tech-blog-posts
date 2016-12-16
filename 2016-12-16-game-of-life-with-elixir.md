---
layout: post
title:  "Using Elixir and Elm for Conway's Game of Life"
date:   2016-12-02 12:54:57 +0100
published: false
---

# Conway's Game of Life using Elixir and Elm

Myself and some colleagues recently used Conway's Game of Life [link] as a way to explore using Elixir, Phoenix and Elm.  Here I will discuss some of the design choices we took and the outcome.

# Why Game of Life?

Although the rules of Game of Life are very simple in that whether a cell survives, dies or is generated in the next generation of the game only depends on it's immediate eight surrounding cells, scaling it to run concurrently and producing a nice web based visualisation are non-trivial problems.

The problem can easily be split into three distinct parts, thus allowing three teams to work on each part almost independently once the interfaces are agreed, with each part exploring different aspects of Elixir and/or Elm.

## Cell Calculation Algorithm

This, at its simplest, is a function that given a grid of cells, each of which has an on/off state, can return the grid of cells for the next generation of the simulation.  Here the challenge lies in developing this using a functional programming language and designing a efficient way of representing the cells and calculating the next generation.

## Parallelising and co-ordinating the problem

The most obvious way to scale the simulation, and the approach that we took, is to split the grid of cells into multiple rectangular 'boards' that represent a subset of cells.  As a cell is influenced by its immediate surrounding eight cells, it is necessary for a board to transfer the state of the cells on its border to its neighbouring eight boards.  That way the neighbouring boards can correctly calculate the state of the next generation cells on their borders.  Keeping track of all the boards, their relative positions to each other and the transmission of border cells between them concurrently is a problem ideally suited to the libraries offered in OTP such as Supervisor, GenServer and GenEvent.

## Visualisation

The requirement here was to provide a nice web UI which would play the game simulation in real time showing the changing cells on the screen and allowing the user to stop, start and change the speed of the simulation and also to view which portion (or board) of the overall grid of cells they wished to view.  Although changes that control the simulation flow from the browser to the web server, and could match a typical HTTP request/response cycle, updates to the state of the cells flow only from the server to the browser and do not fit this pattern.  Therefore utilising Phoenix channels (which sit on top of web-sockets) ideally fit this model.  Furthermore developing a single-page web to handle this communication ideally fits Elm, which also has libraries that communicate easily with Phoenix channels.

## Board Representation

We decided to represent the board using the following Elixir struct:
```
defstruct(
  generation: 0, # iteration number
  origin: {0,0}, # this is the bottom left corner of the board
  size: {5,5}, # size of board {x,y}
  alive_cells: %MapSet{}, # set of tuples of alive cells e.g. [{1,1}, {5,2}].  Bottom left is 0,0
  foreign_alive_cells: %MapSet{}, # set of tuples of alive cells
)
```

Perhaps the most obvious representation is a two dimensional list or map with one entry per cell, and each entry being a boolean to indicate if the cell is alive or dead.  However, the rules of Game of Life mean that the majority of cells are dead and hence a more efficient data structure, at least in terms of memory consumption is just storing which cells are currently alive.  Thus we store a set of tuples where each tuple is the x and y co-ordinates of an alive cell. e.g. `%MapSet{{1, 2}, {3, 4}}`.  We use a set (and not a list or map) to avoid having to deal manually with duplicates.

So as to distinguish between cells that we are responsible for calculating and those that come from the neighbouring boards and we only use for the calculation of our own cells, we store them in a separate set, `foreign_alive_cells`.  Getting all the cells can be simply done by merging the two sets  using `MapSet.union/2`.

As we have multiple boards, we store the co-ordinates of the bottom left corner also as a {x,y} tuple.

## Next Generation Calculation

Calculating the next generation of cells (once we have the cells from the neighbouring boards in `foreign_alive_cells`) can then be done by:
1. Merging `foreign_alive_cells` and `alive_cells` with `MapSet.union/2`
2. Calculating the set of cells that will survive to the next generation (those having 2 or 3 neighbours)
3. Calculating the set of cells that will become alive in the next generation (those having 3 neighbours exactly)
4. Merging the sets of (2) and (3) together

Efficient algorithms for this are already a solved problem in the sphere of the mathematics of cellular automations such as Game of Life, and without doubt, our algorithm could be more efficient, but it is easy to understand and solves it in a purely functional way.  It would be also nice if we could refactor this if we have time to make the code more intuitive and more 'Elixirish'.

## Storing the board state

Although we now have an algorithm for calculating the next board state from the previous generation, we need to store the board itself allow interactions via an API.  For this we provided a thin GenServer API `BoardServer` on top of the board itself.  The only state the the GenServer has is the board state itself:
```
defmodule GameOfLife.BoardServer do
  alias GameOfLife.Board
  use GenServer

  defstruct board: %Board{}
  ...
end
```

API functions tell the board to generate its next state, return its current state, or update its `foreign_alive_cells` given a set of cells from a neighbouring board.

# The Grid

As mentioned previously, we wanted to parallelise the simulation and thus we split the overall grid of cells into multiple boards.  In the grid we store a list of boards, with each board having an id which is a tuple for its bottom left co-ordinates.  In order to communicate with each board (whose state is held in a separate `BoardServer` we need to store its PID in a map of board_ids to PIDs. e.g. `%{ {0,0} => #PID<0.61.0>, {50,50} => #PID<0.62.0> }`

The Grid is also a GenServer providing API functions to add a board and get a list of boards.

# The Ticker

As the Game of Life is turn based, we knew we needed something to co-ordinate the turn of each board so they all remained in sync and on the same generation.  Our initial, and probably overly ambitious thought, was to allow the boards to control themselves just listening for a heartbeat message  published to every board (every 500ms for example) from a central "ticker" to telling them to run the next generation.  As well as providing regular heartbeats the ticker would also provide an API for stopping and starting the heartbeat and controlling its interval.  This again would be a GenServer.

However, we realised that this would lead to race conditions and co-ordination problems if the interval was too short and one or more boards had not completed their next generation calculations before the next heartbeat arrived.  So a simpler solution was just to provide a central run function which instructed boards to run their next generation:

1. Tell all boards to run their next turn
2. Sleep for the interval configured in the ticker
3. Repeat

Therefore although the ticker still provides and API for stopping and starting and controlling its interval, it no longer provides heartbeats, and can simply be used to query how long the currently configured interval is.

# Synchronising boards with their neighbours

To avoid a board having to be aware of its neighbours we instead decided to leverage the functionality of GenEvent.  This provides a pubsub implementation allow boards to publish their state and neighbouring boards to listen for updates.  Creating a central event manager is very simple:
```
defmodule GameOfLife.EventManager do
  use GenEvent

  def start_link, do: {:ok, _pid} = GenEvent.start_link(name: __MODULE__)
end
```

Then once a board has completed calculating its next generation it can publish its new state to the EventManager:
```
GenEvent.notify(GameOfLife.EventManager, {:board_update, board})
```

And neighbouring boards can subscribe to the EventManager using board updates: `GenEvent.add_handler/3` and then act on the `:board_udpate` event:
```
def handle_event({:board_update, neighbour_board}, state) do
 # Update the foreign cells in own board with neighbour_board cells that are on the corresponding edge
 ...
end
```


* Learn about Elm
* Non-trivial project
* Can be split into multiple teams
* Explore Elixir
  * Functional problem
  * OTP for concurrency
  * Phoenix channels with push notifications
  * Single Page App suits this

# Overall design
* Split overall grid of cells into boards
* Each board only needs to communicate border to other boards
* Publish border cells to other boards
* Ticker
** Initially sending heartbeat
