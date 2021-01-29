---
title: "Golang Project: Tic Tac Toe"
date: 2021-01-29 09:23:36 +0800
categories: [Execises, Golang]
tags: [golang, project]
---

In this article, I'll go through my process of writing a simple Tic-Tac-Toe
 game in Golang. Error handling, closures, iota as well as other golang features
 are used to create the game.

> Before we start:
> 
> After some more research to the language, It seems that Go is really not _that_
> different from order C-family programming languages. So I decided that it would
> not be worthwhile to document every single detail of Golang in my blog posts, since
> there's a lot of resources readily available on the internet which go much deeper
> into the Go language features than my blog will ever be able to do. So it doesn't 
> make much sense for me to write about everything, a quick Google search will usually
> work better.
>     
> As a result, this article series will now be focused on my experiences of making experimental
> and/or actual project with Golang. The goal is to show Golang's features and neuances.

[Full code on Github](https://github.com/Miigon/full-codes-from-blog/blob/master/tic-tac-toe.go)

# Define the board and game state

The first thing we want to do is define the board on which the game will be played on.

The game Tic-Tac-Toe has a 3*3 board. Each of the squares can be one of three states:
it can be a `X` or a `O`, or it can be just empty.
We define these states with type alias and const definition, which is Go's equivalance
of `enum`:
```go
// what kind of piece is on a certain square
type squareState int

const (
	none   = iota
	cross  = iota
	circle = iota
)

```

`iota` represents successive integer constants 0,1,2,....  
It resets to 0 whenever the keyword `const` appears in the source code and increments 
after each const specification.  
In our case, `none` would be 0, `cross` would be 1 and `circle` would be 2.

Noted that all the `iota`(except for the first one) can be omitted and still have the same
 effect.

With `suqareState` we can represent the state of one square, now we can use a 3*3 array to
 represent the whole board.
```go
var board [3][3]squareState
```

For each turn, we also need to know whose turn it is. So we introduce another variable to
 represent that:
```go
type player int

var turnPlayer player
```
We don't have to define constants again, the constants defined for the squareState before can be used.

> This is when we run into the discussion about whether Golang's decision to not include
> C/C++ style enum keyword had made the language difficult to use in some cases.
>
> Some have argued that the lack of compile-time type checking makes the code more prone to
> mistakes. `const` names being in the same package scope also can cause some confusion.
> Since it's not always immediately obvious which enum a const name belongs to when you see one.

The last thing for us to do is wrap them (the board & whose turn is it) up as the current 
game state struct:
```go
// current state of the game
type gameState struct {
	board [3][3]squareState
	turnPlayer player
}
```
Storing them as saparate global variables will work as well, but using `struct` is usually a better
practice.  
It makes the code easier to understand. And also enables us to do some other cool things.  
(eg. an "undo" feature, which we will talk about later)

# Draw the board

Next, we need a way to show our board on the screen. We use `fmt` to draw the board to
 the terminal.  

The code is pretty standard:
```go
// define a method for struct type `gameState`
func (state *gameState) drawBoard() {
	for i, row := range state.board {
		for j, square := range row {
			fmt.Print(" ")
			switch square {
			case none:
				fmt.Print(" ")
			case cross:
				fmt.Print("X")
			case circle:
				fmt.Print("O")
			}
			if j != len(row)-1 {
				fmt.Print(" |")
			}
		}
		if i != len(state.board)-1 {
			fmt.Print("\n------------")
		}
		fmt.Print("\n")
	}
}
```
(Notice how `break` is not required in switch cases in Go.)

Now we test our `drawBoard` function with a main function like this:
```go
func main() {
	state := gameState{}

	state.board[0][1] = cross
	state.board[1][1] = circle

	state.drawBoard()
}
```

which yields:

```
$ go run main.go 
   | X |  
------------
   | O |  
------------
   |   |  
```

**Hooray! It works.**

# Game logic

Now it's time for the game logic.

The rule of tic-tac-toe is really simple. The whole game can be discribed as:
```go
for {
	draw_the_board()

	row, column := enter_position_to_place_mark()
	place_mark(row, column)

	if any_one_has_won_the_game() == true {
		break
	}

	next_turn()
}

```

## Placing mark

So far we can already draw the board, but we don't really have a **proper way** to place a mark
 on a square.  
When we are testing our `drawBoard` function, we modified the `board` field of gameState directly,
 but that's not good enough for our game logic. We need it to be able to do a little bit more, eg.
 checking if a square has already had a mark on it.

Therefore we write a function to do just that:
```go
// define error types
type markAlreadyExistError struct {
	row    int
	column int
}

type positionOutOfBoundError struct {
	row    int
	column int
}

// implement Error()
func (e *markAlreadyExistError) Error() string {
	return fmt.Sprintf("position (%d,%d) already has a mark on it.", e.row, e.column)
}

func (e *positionOutOfBoundError) Error() string {
	return fmt.Sprintf("position (%d,%d) is out of bound.", e.row, e.column)
}

// place a mark at a certain position
func (state *gameState) placeMark(row int, column int) error {
	if row < 0 || column < 0 || row >= len(state.board) || column >= len(state.board[row]) {
		return &positionOutOfBoundError{row, column}
	}
	if state.board[row][column] != none {
		return &markAlreadyExistError{row, column}
	}

	state.board[row][column] = squareState(state.turnPlayer) // the actual "placing"
	return nil // no error
}
```
You can see that aside from error handling, the code above really does nothing more than
 changing `state.board[row][column]`.  
However, in real projects, as the project grow more and more complex, instead of directly setting the value
everywhere, __using a function to do it allows for more flexibility and would pay off in the long run__.

The code above defined two new error types `markAlreadyExistError` and `positionOutOfBoundError`.
For them to be considered an `error` type, their respective `Error()` function must be implemented.
That's called _composition_ (compared to _inheritance_).

## Switching turn

Now we can place a mark on a square, what do we need next?

Well, the game has 2 players and they take turns to place marks. So after a mark is placed,
 we would like to switch whose mark will be placed on the next `placeMark()`

We do that by adding a `nextTurn()` function, which sets `state.turnPlayer` to the other player.

```go
type gameResult int

const (
	noWinnerYet = iota
	crossWon
	circleWon
	draw
)

func (state *gameState) whosNext() player {
	return state.turnPlayer
}

func (state *gameState) nextTurn() {
	if state.turnPlayer == cross {
		state.turnPlayer = circle
	} else {
		state.turnPlayer = cross
	}
}

```
`whosNext()` and `nextTurn()` are pretty straightforward and does exactly what it says to do.

## Checking for winner

So for each and every single turn, we need to check if anyone has won the game by placing a mark.
This is when we get into a more interesting part of this project.

### The naive approach

You might be tempted to do something like this:

```go
func (state *gameState) checkForWinner() gameResult {
	// check vertical
	if state.board[0][0] == state.board[0][1] &&
		state.board[0][1] == state.board[0][2] &&
		state.board[0][2] != none {
		return gameResult(state.board[0][0])
	}
	if state.board[1][0] == state.board[1][1] &&
		state.board[1][1] == state.board[1][2] &&
		state.board[1][2] != none {
		return gameResult(state.board[1][0])
	}
	if state.board[2][0] == state.board[2][1] &&
		state.board[2][1] == state.board[2][2] &&
		state.board[2][2] != none {
		return gameResult(state.board[2][0])
	}

	// check horizontal
	if state.board[0][0] == state.board[1][0] &&
		state.board[1][0] == state.board[2][0] &&
		state.board[2][0] != none {
		return gameResult(state.board[0][0])
	}
	if state.board[0][1] == state.board[1][1] &&
		state.board[1][1] == state.board[2][1] &&
		state.board[2][1] != none {
		return gameResult(state.board[0][1])
	}
	if state.board[0][2] == state.board[1][2] &&
		state.board[1][2] == state.board[2][2] &&
		state.board[2][2] != none {
		return gameResult(state.board[0][2])
	}

	// check diagonal
	// ...
	
	return noWinnerYet
}
```

This DOES work, but it's a naive way of doing it. It's long, verbose, and hard to modify.
The board can also only be 3x3 and can not be expanded easily.

### Using for-loops

A way better solution would be to use loops:

```go
func (state *gameState) checkForWinner() gameResult {
CheckHorizontal:
	for _, row := range state.board {
		var lastSquare squareState = row[0]
		for _, square := range row {
			if square != lastSquare {
				continue CheckHorizontal // continue with label, affects the outer loop instead of the inner one
			}
			lastSquare = square
		}
		if lastSquare == cross {
			return crossWon
		} else if lastSquare == circle {
			return circleWon
		}
	}

	// check for verticals and diagonals...

	return noWinnerYet
}

// (the complete code will be about 4 times as long as the code shown)

```
By doing it like this, we can handle board of any dimension. But due to the way the board is
 stored(`board[row][column]`), checking vertical lines using nested for-loops isn't as intuitive as
 checking horizontal lines. And it gets even trickier when it comes to checking for diagonal lines.

Also, the function is still repetitive since the actual "checking" part inside each loop:

```go
		if lastSquare == cross {
			return crossWon
		} else if lastSquare == circle {
			return circleWon
		}
```

are the same.

### 

In order to know if anyone has won the game, we have to check for 3 horizontal lines, 3 vertical
 lines and 2 diagonal lines. We can think of the process of checking each line like this:

1. for each iteration, check if `next square(x + a, y + b)` has the same mark as the `current square(x, y)`
2. set current square position to `(x + a, y + b)`
3. repeat the process until different marks between iteration was found or a border was hit.

It's a generalized description of all the for-loops we discussed before. By using different delta `(a, b)`,
 we can control how we move between different iterations. So __the same code can be used to check for
 horizontals, verticals	and diagonals__.

This is the implementation:

First we define a lambda function for checking one line (can be either horizontal, vertical or diagonal):

```go
checkLine := func(startRow int, startColumn int, deltaRow int, deltaColumn int) gameResult {
	var lastSquare squareState = state.board[startRow][startColumn]
	row, column := startRow+deltaRow, startColumn+deltaColumn

	// loop starts from the second square(startRow + deltaRow, startColumn + deltaColumn)
	for row >= 0 && column >= 0 && row < boardSize && column < boardSize {

		// there can't be a winner if a empty square is present within the line
		if state.board[row][column] == none {
			return noWinnerYet
		}

		if lastSquare != state.board[row][column] {
			return noWinnerYet
		}

		lastSquare = state.board[row][column]
		row, column = row+deltaRow, column+deltaColumn
	}

	// someone has won the game
	if lastSquare == cross {
		return crossWon
	} else if lastSquare == circle {
		return circleWon
	}

	return noWinnerYet
}
```

Then we put it inside our `checkForWinner()` function, alongside some other code to utilize it.

```go
func (state *gameState) checkForWinner() gameResult {
	boardSize := len(state.board) // assuming the board is always square-shaped.

	// define a lambda function for checking one single line
	checkLine := func(startRow int, startColumn int, deltaRow int, deltaColumn int) gameResult {
		// ...
	}

	// check horizontal rows
	for row := 0; row < boardSize; row++ {
		if result := checkLine(row, 0, 0, 1); result != noWinnerYet {
			return result
		}
	}
	// check vertical columns
	for column := 0; column < boardSize; column++ {
		if result := checkLine(column, 0, 0, 1); result != noWinnerYet {
			return result
		}
	}
	// check top-left to bottom-right diagonal
	if result := checkLine(0, 0, 1, 1); result != noWinnerYet {
		return result
	}
	// check top-right to bottom-left diagonal
	if result := checkLine(0, boardSize-1, 1, -1); result != noWinnerYet {
		return result
	}
	// check for draw
	for _, row := range state.board {
		for _, square := range row {
			if square == none {
				return noWinnerYet
			}
		}
	}
	// if no one wins yet, but none of the squares are empty
	return draw
}

```
The code above uses the same `checkLine()` routine to check for horizontals, verticals and diagonals.

# Putting everything together

Now that we can draw the board, place a mark, switch turns and check for potiential winners, it's time
 to put everything together.

```go
func (e player) String() string {
	switch e {
	case none:
		return "none"
	case cross:
		return "cross"
	case circle:
		return "circle"
	default:
		return fmt.Sprintf("%d", int(e))
	}
}

func main() {
	state := gameState{}
	state.turnPlayer = cross // cross goes first

	var result gameResult = noWinnerYet

	// the main game loop
	for {
		fmt.Printf("\nnext player to place a mark is: %v\n", state.whosNext())

		// 1. draw the board onto the screen
		state.drawBoard()

		fmt.Printf("where to place a %v? (input row then column, separated by space)\n> ", state.whosNext())

		// 2. use a loop to take input
		for {
			var row, column int
			fmt.Scan(&row, &column)

			e := state.placeMark(row-1, column-1) // -1 so coordinate starts at (1,1) instead of (0,0)

			// if a valid position was entered, break out from the input loop
			if e == nil {
				break
			}

			// if an invalid position was entered, prompt the player to re-enter another position
			fmt.Println(e)
			fmt.Printf("please re-enter a position:\n> ")
		}

		// 3. check if anyone has won the game
		result = state.checkForWinner()
		if result != noWinnerYet {
			break
		}

		// 4. if no one has won in this turn, go on for next turn and continue the game loop
		state.nextTurn()

		fmt.Println()
	}

	state.drawBoard()

	switch result {
	case crossWon:
		fmt.Printf("cross won the game!\n")
	case circleWon:
		fmt.Printf("circle won the game!\n")
	case draw:
		fmt.Printf("the game has ended with a draw!\n")
	}
}
```

Noted that we defined method `String()` for `player` type at the beginning.

This is done for the following line of code to work.
```go
fmt.Printf("\nnext player to place a mark is: %v\n", state.whosNext())
```

Defining this method makes type `player` a _Stringer_, meaning something that has a `String()` method and
 can be converted into a string.

In this case, `fmt.Printf` uses the method internally to convert a `player` into a string,
 then prints it out.

# Testing the game

Now we compile and run the game:
```
next player to place a mark is: cross
   |   |  
------------
   |   |  
------------
   |   |  
where to place a cross? (input row then column, separated by space)
> 1 1

next player to place a mark is: circle
 X |   |  
------------
   |   |  
------------
   |   |  
where to place a circle? (input row then column, separated by space)
> 1 2

...

next player to place a mark is: cross
 X | O | X
------------
 O | O | X
------------
   | X | O
where to place a cross? (input row then column, separated by space)
> 3 1
 X | O | X
------------
 O | O | X
------------
 X | X | O
the game has ended with a draw!
```

From another run:

```
...

next player to place a mark is: circle
 X | X | O
------------
 X | O |  
------------
   |   |  
where to place a circle? (input row then column, separated by space)
> 3 1

 X | X | O
------------
 X | O |  
------------
 O |   |  
circle won the game!
```

**There you have it, a fully working tic-tac-toe in Go.**
