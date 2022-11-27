---
layout: single
title:  "Conway's Game of Life in JavaScript"
date:   2021-10-04 18:40:03 -0600
categories: javascript node
---

[Conway's Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life) is a simple 2D cell-growth simulation that follows four simple rules, which are as follows:

1. Any live cell with fewer than two live neighbours dies, as if by underpopulation
2. Any live cell with two or three live neighbours lives on to the next generation
3. Any live cell with more than three live neighbours dies, as if by overpopulation
4. Any dead cell with exactly three live neighbours becomes a live cell, as if by reproduction

## The Game of Life in JavaScript

Let's start from the top level and work our way down through my Game of life solver. First off is the *run* function which simply takes in a starting board of cells and returns that board after a specified number of generations have elapsed.

```JavaScript
export const run = (startingBoard, numGenerations) => {
    let currentBoard = startingBoard;
    
    for (let i = 0; i < numGenerations; ++i) {
        currentBoard = solveGeneration(currentBoard); 
    }

    return currentBoard;
};
```

*solveGeneration* first determines which cells should die in the following generation, finds all dead cells adjacent to live cells, revives any dead cells with three live neighbors, and then combines the lists of survived and revived cells

```JavaScript
export const solveGeneration = (board) => {
    let survivedCells = killCells(board);

    let deadNeigborCells = getAdjacentDeadCells(board);

    let revivedCells = getRevivedCells(deadNeigborCells, board);

    return survivedCells.concat(revivedCells);
};
```

*killCells* simulates rules one and three by filtering out any cells from a given board with zero, one, or four or more neighbors

```JavaScript
export const killCells = (liveCells) => {
    return liveCells.filter(c => numNeighbors(c, liveCells) > 1 && 
                                 numNeighbors(c, liveCells) < 4
    );
};
```

The return statement in *areNeighbors* only return true if the two input cells are horizontally, vertically, or diagonally adjacent to each other. *numNeighbors* filters out any cells not adjacent to a given cell and returns their count.

```JavaScript
export const numNeighbors = (cell, board) => {
    return board.filter(c => areNeighbors(cell, c)).length;
};

export const areNeighbors = (cell1, cell2) => {
    let deltaX = Math.abs(cell1.x - cell2.x);
    let deltaY = Math.abs(cell1.y - cell2.y);

    //Return true if both delta x and y are no greater than one and at least one is equal to 1 
    return (deltaX <= 1 && deltaY <= 1) && (deltaX == 1 || deltaY == 1);
}
```

*getAdjacentCells* iterates through each live cell and gets its neighboring cells, dead or alive. Each neighboring cell is added to a list if it hasn't already, and then that list of cells is returned after any live cells are filtered out of it.  

```JavaScript
export const getAdjacentDeadCells = (liveCells) => {
    let adjacentCells = [];

    liveCells.forEach(liveCell => {
        for (let x = -1; x <= 1; x++) {
            for (let y = -1; y <= 1; y++) {
                let cell = new Cell(liveCell.x - x, liveCell.y - y)
                if (!adjacentCells.some(deadCell => deadCell.x === cell.x && deadCell.y === cell.y)) {
                    adjacentCells.push(cell);
                }
            }
        }
    });


    return adjacentCells.filter(cell => 
                !liveCells.some(liveCell => liveCell.x === cell.x && liveCell.y === cell.y));
    
};
```

*getRevivedCells* simulates rule 4 by returning all cells in a list of dead cells with exactly three live neighbors

```JavaScript
export const getRevivedCells = (deadCells, liveCells) => {
    return deadCells.filter(deadCell => numNeighbors(deadCell, liveCells) == 3)
};
```

## Visualizing the Results

*visualizeBoard* converts a given board of cells to a string. It does so by finding the extremes of the board and iterating over each pair of coordinates within their ranges. For each pair of coordinates, we check to see if a cell is there and append an "O" to the board's string representation if so and a space if not.

```JavaScript
const visualizeBoard = (board) => {

    let boardString = "";

    const minBoardX = Math.min.apply(Math, board.map(cell => cell.x));
    const maxBoardX = Math.max.apply(Math, board.map(cell => cell.x));

    const minBoardY = Math.min.apply(Math, board.map(cell => cell.y));
    const maxBoardY = Math.max.apply(Math, board.map(cell => cell.y)) 


    for (let y = maxBoardY; y >= minBoardY; y--) {
        for (let x = minBoardX; x <= maxBoardX; x++) {

            if (board.some(c => c.x == x && c.y == y)) {
                boardString += "O";
            }
            else {
                boardString += " ";
            }
        }
        boardString += '\n'
    }

    return boardString;
}
```

One simple way of printing out the results of the program is through the browser. To do that, I created a simple express api that calculates the resulting board after a given number of generations from a static board, [the acorn](https://www.youtube.com/watch?v=Aq51GfPmD54).

```JavaScript
const acorn = [ new Cell(1, 1),
                new Cell(2, 1),
                new Cell(2, 3),
                new Cell(4, 2),
                new Cell(5, 1),
                new Cell(6, 1),
                new Cell(7, 1)];

var app = express();

app.get('/board', (req, res) => {
    const numGenerations = req.query.numGenerations;
    const board = run(acorn, numGenerations);
    const boardString = visualizeBoard(board);


    res.end(boardString);
});

var server = app.listen(8080, () => {
    console.log(`Listening at http://${server.address().address}:${server.address().port}`);
})
```

![Acorn Visualization after 100 Generations](https://i.imgur.com/8YaJi5a.png)

## Testing the Program

I've shown all there is to show for my solver, but if you're interested in seeing them, I also have a suite of passing tests. Their presence was invaluable, as manually verifying the results of a program such as this is tedious and error-prone.

```JavaScript
const adjacentCells =[
    new Cell(1, 3), new Cell(2, 3), new Cell(3, 3),
    new Cell(1, 2),                 new Cell(3, 2),
    new Cell(1, 1), new Cell(2, 1), new Cell(3, 1)]

const nonAdjacentCells = [
    new Cell(-2, 2),  new Cell(-1, 2),  new Cell(0, 2),  new Cell(1, 2),  new Cell(2, 2),
    new Cell(-2, 1),                                                      new Cell(2, 1),
    new Cell(-2, 0),                                                      new Cell(2, 0),
    new Cell(-2, -1),                                                     new Cell(2, -1),
    new Cell(-2, -2), new Cell(-1, -2), new Cell(0, -2), new Cell(1, -2), new Cell(2, -2),
]



test.each(adjacentCells)('areNeighbors returns true when given two adjacent cells', (adjacentCell) => {
    expect(areNeighbors(new Cell(2, 2), adjacentCell)).toBe(true);
});


test.each(nonAdjacentCells)('areNeighbors returns false when given two non-adjacent cells', ( cell2) => {
    expect(areNeighbors(new Cell(0, 0), cell2)).toBe(false);
});


test('numNeighbors returns eight for cell surrounded by 8 cells', () => {
    expect(numNeighbors(new Cell(2, 2), adjacentCells)).toBe(8);              
});


test('numNeighbors returns 0 for cell with no adjacent neighbors', () => {
    expect(numNeighbors(new Cell(0, 0), nonAdjacentCells)).toBe(0);
});


test('getAdjacentDeadCells returns correct cells for 3x1 strip of cells', () => {
    const startingBoard = [new Cell(1, 1), new Cell(2, 1), new Cell(3, 1)].sort();

    const expectedCells = new Set([
        new  Cell(0, 2), new Cell(1, 2), new Cell(2, 2), new Cell(3, 2), new Cell(4, 2),
        new  Cell(0, 1),                                                 new Cell(4, 1),
        new  Cell(0, 0), new Cell(1, 0), new Cell(2, 0), new Cell(3, 0), new Cell(4, 0),
    ]);

    const actualCells = new Set(getAdjacentDeadCells(startingBoard));

    expect(actualCells).toEqual(expectedCells);
});


test('getAdjacentDeadCells returns empty array for 0 cells', () => {
    const expectedBoard = [];
    const actualBoard = getAdjacentDeadCells([]);

    expect(actualBoard).toEqual(expectedBoard);
});


test('killCells removes cells with 0 neighbors', () => {
    const startingBoard = [new Cell(1, 1)];
    const expectedBoard = [];

    const actualBoard = killCells(startingBoard);

    expect(actualBoard).toEqual(expectedBoard);
});

test('killCells removes cells with 1 neighbor', () => {
    const startingBoard = [new Cell(1, 1), new Cell(1, 2)];
    const expectedBoard = [];

    const actualBoard = killCells(startingBoard);

    expect(actualBoard).toEqual(expectedBoard);
});


test('killCells removes cells with 4 neighbors', () => {
    const startingBoard = [
                        new Cell(0, 1),
        new Cell(-1, 0), new Cell(0, 0), new Cell(1, 0),
                         new Cell(0, -1)
    ];

    const expectedBoard = new Set([
                          new Cell(0, 1),
        new Cell(-1, 0),                 new Cell(1, 0),
                          new Cell(0, -1)
    ]); 

    const actualBoard = new Set(killCells(startingBoard));

    expect(actualBoard).toEqual(expectedBoard);
});


test('Any adjacent cell with fewer than two neighbors dies as if by underpopulation', () => {
    const startingBoard = [new Cell(1, 1)];
    const expectedBoard = [];
    
    const actualBoard = solveGeneration(startingBoard);


    expect(actualBoard).toEqual(expectedBoard);
});


test('Any live cell with two or three neighbors lives on to the next generation', () => {
    const startingBoard = [new Cell(1, 1), new Cell(2, 1), new Cell(1, 2)];
    const expectedBoard = new Set( [
        new Cell(1, 2), new Cell(2, 2), 
        new Cell(1, 1), new Cell(2, 1)
    ]);

    const actualBoard = new Set(solveGeneration(startingBoard));



    expect(actualBoard).toEqual(expectedBoard);
});


test('Any live cell with more than three neighbors dies, as if by overpopulation', () => {
    const startingBoard = [
        new Cell(2, 4), new Cell(3, 4), new Cell(4, 4),
        new Cell(2, 3), new Cell(3, 3), new Cell(4, 3),
        new Cell(2, 2), new Cell(3, 2), new Cell(4, 2),
    ];


    const expectedBoard = new Set([
                                     new Cell(3,5),
                    new Cell(2, 4),                 new Cell(4, 4),
    new Cell(1, 3),                                                 new Cell(5, 3),
                    new Cell(2, 2),                 new Cell(4, 2),
                                    new Cell(3, 1)
    ]);

    const actualBoard = new Set(solveGeneration(startingBoard));


    expect(actualBoard).toEqual(expectedBoard);
})


test ('Any dead cell with exactly three live neighbors becomes a live cell, as if by reproduction', () => {
    const startingBoard = [new Cell(1, 1), new Cell(3, 1), new Cell(1, 3)]
    const expectedBoard = [new Cell(2, 2)];
    
    const actualBoard = solveGeneration(startingBoard);

    expect(actualBoard).toEqual(expectedBoard);
})
```
