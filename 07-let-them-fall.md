<center>![](http://bloc-books.s3.amazonaws.com/swiftris/08-let-them-fall-final.gif)</center>

>“And why do we fall, Bruce? So we can learn to pick ourselves up.”<br>
>-- Thomas Wayne

## Let Them Fall

We've worked pretty hard on preparing our blocks and shapes, let's just make them fall already! Thankfully, this only requires a bit of meaningful code before it can be witnessed in Swiftris! Let's begin by adding some helper methods to `Shape.swift` which will allow us to establish and alter our shape's location:

```objc(Shape.swift)
    final func initializeBlocks() {
        if let blockRowColumnTranslations = blockRowColumnPositions[orientation] {
            for i in 0..<blockRowColumnTranslations.count {
                let blockRow = row + blockRowColumnTranslations[i].rowDiff
                let blockColumn = column + blockRowColumnTranslations[i].columnDiff
                let newBlock = Block(column: blockColumn, row: blockRow, color: color)
                blocks.append(newBlock)
            }
        }
    }

+    final func rotateBlocks(orientation: Orientation) {
+        if let blockRowColumnTranslation:Array<(columnDiff: Int, rowDiff: Int)> = blockRowColumnPositions[orientation] {
// #1
+            for (idx, (columnDiff:Int, rowDiff:Int)) in enumerate(blockRowColumnTranslation) {
+                blocks[idx].column = column + columnDiff
+                blocks[idx].row = row + rowDiff
+            }
+        }
+    }

+    final func lowerShapeByOneRow() {
+        shiftBy(0, rows:1)
+    }

// #2
+    final func shiftBy(columns: Int, rows: Int) {
+        self.column += columns
+        self.row += rows
+        for block in blocks {
+            block.column += columns
+            block.row += rows
+        }
+    }

// #3
+    final func moveTo(column: Int, row:Int) {
+        self.column = column
+        self.row = row
+        rotateBlocks(orientation)
+    }

+    final class func random(startingColumn:Int, startingRow:Int) -> Shape {
+        switch Int(arc4random_uniform(NumShapeTypes)) {
// #4
+        case 0:
+            return SquareShape(column:startingColumn, row:startingRow)
+        case 1:
+            return LineShape(column:startingColumn, row:startingRow)
+        case 2:
+            return TShape(column:startingColumn, row:startingRow)
+        case 3:
+            return LShape(column:startingColumn, row:startingRow)
+        case 4:
+            return JShape(column:startingColumn, row:startingRow)
+        case 5:
+            return SShape(column:startingColumn, row:startingRow)
+        default:
+            return ZShape(column:startingColumn, row:startingRow)
+        }
+    }
```

At **#1** we introduce the `enumerate` operator. This allows us to iterate through an array object by defining an index variable - `idx` - as well as the contents at that index: `(columnDiff:Int, rowDiff:Int)`. This saves us the added step of recovering it from the array, `let tuple = blockRowColumnTranslation[idx]`. We loop through the blocks and assign them their row and column based on the translations provided by the Tetromino subclass.

At **#2**, we've included a simple `shiftBy(columns: Int, rows: Int)` method which will adjust each row and column by `rows` and `columns`, respectively.

At **#3** we provide an absolute approach to position modification by setting the `column` and `row` properties before rotating the blocks to their current orientation which causes an accurate realignment of all blocks relative to the new `row` and `column` properties.

At **#4** we've created a method to generate a random Tetromino shape and you can see that subclasses naturally inherit initializers from their parent class.

We'll need a class that manages Swiftris' game logic, the brains behind the entire Swiftris operation. We could name this something clever like `GameMaster`, `Blocketeer`, or `TetrominosPizza`, but instead, we'll create an anticlimactic file named `Swiftris.swift` and replace its contents with the following:

```objc(Swiftris.swift)
-import Foundation
// #1
+let NumColumns = 10
+let NumRows = 20

+let StartingColumn = 4
+let StartingRow = 0

+let PreviewColumn = 12
+let PreviewRow = 1

+class Swiftris {
+    var blockArray:Array2D<Block>
+    var nextShape:Shape?
+    var fallingShape:Shape?

+    init() {
+        fallingShape = nil
+        nextShape = nil
+        blockArray = Array2D<Block>(columns: NumColumns, rows: NumRows)
+    }

+    func beginGame() {
+        if (nextShape == nil) {
+            nextShape = Shape.random(PreviewColumn, startingRow: PreviewRow)
+        }
+    }

// #2
+    func newShape() -> (fallingShape:Shape?, nextShape:Shape?) {
+        fallingShape = nextShape
+        nextShape = Shape.random(PreviewColumn, startingRow: PreviewRow)
+        fallingShape?.moveTo(StartingColumn, row: StartingRow)
+        return (fallingShape, nextShape)
+    }
+}
```

`Swiftris` looks simple for now, but don't worry, things will get messy soon. For now, `Swiftris` maintains a handful of important constants and a couple methods which `GameViewController` will find useful. At **#1** we've defined the total number of rows and columns on the game board, the location of where each piece starts and the location of where the preview piece belongs.

At **#2**, we have a method which assigns `nextShape`, our preview shape,  as `fallingShape`. `fallingShape`  is the moving Tetromino. `newShape()` then creates a new preview shape before moving `fallingShape` to the starting row and column. This method returns a tuple of optional `Shape` objects - we'll see why in a later checkpoint.

It's time to work with visuals again, let's dig into precisely how we're going to display these pieces on screen by adding a few methods to `GameScene.swift`

```objc(GameScene.swift)
import SpriteKit

// #1
+let BlockSize:CGFloat = 20.0

 let TickLengthLevelOne = NSTimeInterval(600)

class GameScene: SKScene {
// #2
+   let gameLayer = SKNode()
+   let shapeLayer = SKNode()
+   let LayerPosition = CGPoint(x: 6, y: -6)

    var tick:(() -> ())?
    var tickLengthMillis = TickLengthLevelOne
    var lastTick:NSDate?

+   var textureCache = Dictionary<String, SKTexture>()

    required init(coder aDecoder: NSCoder) {
        fatalError("NSCoder not supported")
    }

    override init(size: CGSize) {
        super.init(size: size)

        anchorPoint = CGPoint(x: 0, y: 1.0)

        let background = SKSpriteNode(imageNamed: "background")
        background.position = CGPoint(x: 0, y: 0)
        background.anchorPoint = CGPoint(x: 0, y: 1.0)
        addChild(background)

+       addChild(gameLayer)

+       let gameBoardTexture = SKTexture(imageNamed: "gameboard")
+       let gameBoard = SKSpriteNode(texture: gameBoardTexture, size: CGSizeMake(BlockSize * CGFloat(NumColumns), BlockSize * CGFloat(NumRows)))
+       gameBoard.anchorPoint = CGPoint(x:0, y:1.0)
+       gameBoard.position = LayerPosition

+       shapeLayer.position = LayerPosition
+       shapeLayer.addChild(gameBoard)
+       gameLayer.addChild(shapeLayer)
    }

    override func update(currentTime: CFTimeInterval) {
        /* Called before each frame is rendered */
        if lastTick == nil {
            return
        }
        var timePassed = lastTick!.timeIntervalSinceNow * -1000.0
        if timePassed > tickLengthMillis {
            lastTick = NSDate.date()
            tick?()
        }
    }

    func startTicking() {
        lastTick = NSDate.date()
    }

    func stopTicking() {
        lastTick = nil
    }

// #3
+    func pointForColumn(column: Int, row: Int) -> CGPoint {
+        let x: CGFloat = LayerPosition.x + (CGFloat(column) * BlockSize) + (BlockSize / 2)
+        let y: CGFloat = LayerPosition.y - ((CGFloat(row) * BlockSize) + (BlockSize / 2))
+        return CGPointMake(x, y)
+    }

+    func addPreviewShapeToScene(shape:Shape, completion:() -> ()) {
+        for (idx, block) in enumerate(shape.blocks) {
// #4
+            var texture = textureCache[block.spriteName]
+            if texture == nil {
+                texture = SKTexture(imageNamed: block.spriteName)
+                textureCache[block.spriteName] = texture
+            }
+            let sprite = SKSpriteNode(texture: texture)
// #5
+            sprite.position = pointForColumn(block.column, row:block.row - 2)
+            shapeLayer.addChild(sprite)
+            block.sprite = sprite

+            // Animation
+            sprite.alpha = 0
// #6
+            let moveAction = SKAction.moveTo(pointForColumn(block.column, row: block.row), duration: NSTimeInterval(0.2))
+            moveAction.timingMode = .EaseOut
+            let fadeInAction = SKAction.fadeAlphaTo(0.7, duration: 0.4)
+            fadeInAction.timingMode = .EaseOut
+            sprite.runAction(SKAction.group([moveAction, fadeInAction]))
+       }
+        runAction(SKAction.waitForDuration(0.4), completion: completion)
+    }

+    func movePreviewShape(shape:Shape, completion:() -> ()) {
+        for (idx, block) in enumerate(shape.blocks) {
+            let sprite = block.sprite!
+            let moveTo = pointForColumn(block.column, row:block.row)
+            let moveToAction:SKAction = SKAction.moveTo(moveTo, duration: 0.2)
+            moveToAction.timingMode = .EaseOut
+            sprite.runAction(
+                SKAction.group([moveToAction, SKAction.fadeAlphaTo(1.0, duration: 0.2)]), completion:nil)
+        }
+        runAction(SKAction.waitForDuration(0.2), completion: completion)
+    }

+    func redrawShape(shape:Shape, completion:() -> ()) {
+        for (idx, block) in enumerate(shape.blocks) {
+            let sprite = block.sprite!
+            let moveTo = pointForColumn(block.column, row:block.row)
+            let moveToAction:SKAction = SKAction.moveTo(moveTo, duration: 0.05)
+            moveToAction.timingMode = .EaseOut
+            sprite.runAction(moveToAction, completion: nil)
+        }
+        runAction(SKAction.waitForDuration(0.05), completion: completion)
+    }
 }
```

This is a big change but don't fear, it will all make sense soon. At **#1** we simply define the point size of each block sprite - in our case `20.0 x 20.0` - the lower of the available resolution options for each block image. We also declare a layer position which will give us an offset from the edge of the screen.

At **#2** we've introduced a couple of `SKNode`s which can be thought of as superimposed layers of activity within our scene. The `gameLayer` sits above the background visuals and the `shapeLayer` sits atop that.

At **#3** we've written `GameScene`'s most important function, `pointForColumn(Int, Int)`. This function returns the precise coordinate on the screen for where a block sprite belongs based on its row and column position. The math here looks funky but just know that each sprite will be anchored at its center, therefore we need to find its center coordinate before placing it in our `shapeLayer` object.

At **#4** we've created a method which will add a shape for the first time to the scene as a preview shape. We use a dictionary to store copies of re-usable `SKTexture` objects since each shape will require multiple copies of the same image.

At **#5** we use our convenient `pointForColumn(Int, Int)` method to place each block's sprite in the proper location. We start it at `row - 2`, such that the preview piece animates smoothly into place from a higher location.

At **#6**, we introduce `SKAction` objects which are responsible for visually manipulating `SKNode` objects. Each block will *fade* and *move* into place as it appears as part of the next piece. It will move two rows down and fade from complete transparency to 70% opacity. This small design choice lets the player ignore the preview piece easily if they so choose since it will be duller than the active moving piece. The remaining two methods make use of the same `SKAction` objects to move and redraw each block for a given shape.

Our drawing layer is ready, our logic layer is ready, but now we need to connect the two with our user interface class, `GameViewController`:

```objc(GameViewController.swift)
import UIKit
import SpriteKit

class GameViewController: UIViewController {

    var scene: GameScene!
+   var swiftris:Swiftris!

    override func viewDidLoad() {
        super.viewDidLoad()

        // Configure the view.
        let skView = view as SKView
        skView.multipleTouchEnabled = false

        // Create and configure the scene.
        scene = GameScene(size: skView.bounds.size)
        scene.scaleMode = .AspectFill
// #1
+       scene.tick = didTick

+       swiftris = Swiftris()
+       swiftris.beginGame()

        // Present the scene.
        skView.presentScene(scene)

// #2
+        scene.addPreviewShapeToScene(swiftris.nextShape!) {
+            self.swiftris.nextShape?.moveTo(StartingColumn, row: StartingRow)
+            self.scene.movePreviewShape(self.swiftris.nextShape!) {
+                let nextShapes = self.swiftris.newShape()
+                self.scene.startTicking()
+                self.scene.addPreviewShapeToScene(nextShapes.nextShape!) {}
+            }
+        }
    }

    override func prefersStatusBarHidden() -> Bool {
        return true
    }

// #3
+    func didTick() {
+        swiftris.fallingShape?.lowerShapeByOneRow()
+        scene.redrawShape(swiftris.fallingShape!, completion: {})
+    }
}
```

At **#1** we've set a closure for the `tick` property of `GameScene.swift`. Remember that functions are simply named closures. In our case, we've used a function named `didTick()`. We define `didTick()` at **#3**. All it does is lower the falling shape by one row and then asks `GameScene` to redraw the shape at its new location.

At **#2** we add `nextShape` to the game layer at the preview location. When that animation completes, we reposition the underlying `Shape` object at the starting row and starting column before we ask `GameScene` to move it from the preview location to its starting position. Once that completes, we ask `Swiftris` for a new shape, begin ticking, and add the newly established upcoming piece to the preview area.

Run `Swiftris` and observe the majesty of an ever-falling shape as it exceeds the limits of our imaginary digital boundaries!
