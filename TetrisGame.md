# Retro-Tetris main Game Class
# Eugen Senator

public class TetrisGame : MonoBehaviour {

	// Private Variables
	public float gameSpeed = 0.5f;
	private delegate void SpawnHandler();
	private event SpawnHandler spawnHandler;
	private GameObject shape;
	private Transform[] shapeChildren;
	private int[,] gameBoard;
	private bool gameOver = false;
	private int gameOverHeight = 20;
	private int scoreMultiplier;
	private GameObject nextStone;
	private int nextShape;

	// Public Variables
	public GameObject[] shapes;
	[Range(0, 3)]
	public int rotationStep = 0;
	public bool spawnShape = true;
	public int score = 0;
	public Transform nextStoneSpawner;
	public Button restartButton;
	public Text gameOverText;
	public Transform panelPos;

	// Use this for initialization
	void Start () {
		// Created Spawn Event, for other Spawn Algorithm in Future
		spawnHandler += SpawnShape;
		gameBoard = transform.GetComponent<CreateBoard>().gameBoard;
		score = 0;
		rotationStep = 0;
		nextShape = Random.Range(0, shapes.Length);
		spawnHandler();
		InvokeRepeating("MoveShapeDown", 1, gameSpeed);
		restartButton.interactable = false;
		gameOverText.enabled = false;
	}
	
	// Update is called once per frame
	void Update () {
		if(!gameOver)
		{
			if(spawnShape)
			{
				spawnHandler();
			}

			//Rotate Shape
			if(shape != null)
			{
				ControlShape();
				// Disable Rotation for OBlock
				if(shape.tag != "OBlock")
					RotateShape();

				// Automatic Rotate NextStone Shape
				nextStone.transform.Rotate(new Vector3(0, 1, 0));
			}
		} else
		{
			// If Game Over Enable Restart and Text
			gameOverText.enabled = true;
			restartButton.interactable = true;
		}

		if(Input.GetKeyDown(KeyCode.Escape))
			Application.Quit();
	}

	public void Restart()
	{
		Application.LoadLevel(0);
	}

	//Method for Spawning Shape
	private void SpawnShape()
	{
		if(spawnShape)
		{
			// Create Shape
			shape = GameObject.Instantiate(shapes[nextShape]) as GameObject;
			shapeChildren = shape.transform.GetChildren();

			// Just to be clear, if the Shape scale is other then 1 
			float shapeXScale = shape.transform.localScale.x;
			float shapeYScale = shape.transform.localScale.y;

			// Place them properly on the Board depending on Scale (now its fixed to Scale 1)
			float xPos = gameBoard.GetLength(0) * shapeXScale / 2 - (1 * shapeXScale);
			float yPos = gameBoard.GetLength(1) * shapeYScale - (3 * shapeYScale);

			shape.transform.position = new Vector3(xPos, yPos, shape.transform.position.z - 0.2f);

			//Reset rotationStep every Spawn
			rotationStep = 0;

			// Destroy Old NextStone Shapes
			if(nextStone != null)
				Destroy (nextStone);

			
			// Get Random Number for a Shape
			int randomShape = Random.Range(0, shapes.Length);
			// Instantiate Next Shape for User
			nextStone = GameObject.Instantiate(shapes[randomShape], nextStoneSpawner.position, Quaternion.identity) as GameObject;
			nextStone.transform.position = new Vector3(nextStone.transform.position.x - 5, nextStone.transform.position.y - 4, nextStone.transform.position.z - 10);
			nextStone.transform.localScale *= 4;
			nextShape = randomShape;
		}

		spawnShape = false;
	}

	private void MoveShapeDown()
	{
		if(shape != null && !gameOver)
		{
			// Check if Blocks doesnt hit something on Board
			if(CheckBoard())
			{
				// Move Shape down by 1
				shape.transform.position = new Vector3(shape.transform.position.x, shape.transform.position.y - 1, shape.transform.position.z);
			
			} else {

				// Mark Block Positions on Board as unavailable
				for(int i = 0; i < shapeChildren.Length; i++)
				{
					if(shapeChildren[i] != null)
						gameBoard[Mathf.RoundToInt(shapeChildren[i].position.x), Mathf.RoundToInt(shapeChildren[i].position.y)] = 1;
				}

				// Reset Multiplier
				scoreMultiplier = 0;

				// When positions on the Board are marked
				// Check Rows, if they needed to be deleted
				CheckRow(Mathf.RoundToInt(1));
				CheckRow(gameOverHeight);

				// Spawn new Shape
				spawnShape = true;
			}
		}
	}

	private void CheckRow(int height)
	{
		int blockCount = 0;
		GameObject[] blocks = GameObject.FindGameObjectsWithTag("Blocks");

		for(int i = 1; i < gameBoard.GetLength(0) - 1; i++)
		{
			if(gameBoard[i, height] == 1)
				blockCount++;
		}

		if(height >= gameOverHeight && blockCount > 0){// If the current height is game over height, and there is more than 0 block, then game over
			gameOver = true;
		}

		if(blockCount == 10) // The row is full
		{
			scoreMultiplier++;
			// Start from bottom of the board(withouth edge and block spawn space)
			for(int i = height; i < gameBoard.GetLength(1) - 2; i++)
			{
				for(int j = 1; j < gameBoard.GetLength(0) - 1; j++)
				{
					foreach(GameObject go in blocks){
						
						int yPos = Mathf.RoundToInt(go.transform.position.y);
						int xPos = Mathf.RoundToInt(go.transform.position.x);
						
						if(xPos == j && yPos == i){
							
							if(yPos == height) // The row we need to destroy
							{
								gameBoard[xPos, yPos] = 0; // Set empty space
								Destroy(go);
								
								score += blockCount * scoreMultiplier;
							}
							else if(yPos > height)
							{
								gameBoard[xPos, yPos] = 0; // Set old position to empty
								gameBoard[xPos, yPos - 1] = 1; // Set new position 
								go.transform.position = new Vector3(xPos, yPos - 1, go.transform.position.z);// Move block down
							}
						}
					}
				}
			}
			CheckRow(height);
		}else if(height + 1 < gameBoard.GetLength(1) - 2){
			CheckRow(height + 1); //Check row above this
		}
	}

	// Method for checks if something is beneath the falling blocks
	private bool CheckBoard()
	{
		for(int i = 0; i < shapeChildren.Length; i++)
		{
			// If Blocks are blocked return false
			if(shapeChildren[i] != null)
				if(gameBoard[Mathf.RoundToInt(shapeChildren[i].position.x), Mathf.RoundToInt(shapeChildren[i].position.y - 1)] == 1)
					return false;
		}

		return true;
	}

	// User Control of Shape
	private void ControlShape()
	{
		if(Input.GetKeyDown(KeyCode.LeftArrow))
		{
			if(InsideBoard(false))
				shape.transform.position += Vector3.left;
		}

		if(Input.GetKeyDown(KeyCode.RightArrow))
		{
			if(InsideBoard(true))
				shape.transform.position += Vector3.right;
		}

		if(Input.GetKey(KeyCode.DownArrow) && shape != null)
			MoveShapeDown();
	}

	private void RotateShape()
	{
		if(Input.GetKeyDown(KeyCode.UpArrow))
		{
			rotationStep++;
			shape.transform.eulerAngles = new Vector3(0, 0, rotationStep * 90);
			
			// Reset Rotation if the Block is inside unavailable Space
			for(int i = 0; i < shapeChildren.Length; i++)
			{
				// Rotate backwards
				if(shapeChildren[i] != null)
				{
					if(gameBoard[Mathf.RoundToInt(shapeChildren[i].position.x), Mathf.RoundToInt(shapeChildren[i].position.y)] == 1)
					{
						rotationStep--;
						shape.transform.eulerAngles = new Vector3(0, 0, rotationStep * 90);
					}
				}
			}

			// Reset Rotation Step
			if(rotationStep > 3)
				rotationStep = 0;
		}
	}

	private bool InsideBoard(bool leftOrRight = false)
	{
		for(int i = 0; i < shapeChildren.Length; i++)
		{
			//Check if Left side is blocked
			if(!leftOrRight)
				if(gameBoard[Mathf.RoundToInt(shapeChildren[i].position.x) - 1, Mathf.RoundToInt(shapeChildren[i].position.y)] == 1)
			  		return false;

			//Check if Right side is blocked
			if(leftOrRight)
				if(gameBoard[Mathf.RoundToInt(shapeChildren[i].position.x) + 1, Mathf.RoundToInt(shapeChildren[i].position.y)] == 1)
					return false;
		}

		// Nothing blocked, return true
		return true;
	}
}
