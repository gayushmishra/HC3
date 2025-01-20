# HC3 - A Human-Like Chess Trainer
**Play HC3 here:** [HC3 on Lichess](https://lichess.org/@/HC2_18-20)

When learning a new opening or practicing common middle games, it’s helpful to practice these positions over the board. However, human opponents rarely steer into the specific positions you want to study. A practical alternative is to play against a computer, but even engines designed for specific skill levels make unrealistic, unnatural moves. HC3 aims to replicate **natural human behavior** at a certain skill level, not just the best move.

The most well-known chess engine, **Stockfish**, uses search algorithms to find the move leading to the most favorable position at a given depth. Neural network-based engines like **AlphaZero** and **Leela Chess Zero** use self-play reinforcement learning to optimize move choice for winning. More recently, **Maia**, built with a Leela-like CNN architecture, learned from real online Lichess games rather than self-play. As a result, Maia’s moves were not always the best but instead reflected the most natural response for each position.

## Key Features

**HC3** builds on the Maia chess engine with four significant improvements:

1. **Transformer Architecture**: While Maia uses a CNN architecture originally inspired by Leela, HC3 adopts a ViT-like architecture from newer generations of Leela.
2. **Secondary Model Inputs**: In addition to the raw board state, HC3 directly incorporates supplementary information that could otherwise be inferred from the board. This approach avoids wasting valuable parameters by eliminating the need to derive this information internally.
3. **Position Metadata**: HC3 integrates general game information, such as each player’s ELO, time control, and increment. This additional context enhances the model’s understanding beyond just the board.
4. **Clock Integration**: The clock is pivotal in shaping human-like chess. HC3 receives information on both its own and its opponent’s remaining time for each position. It also learns to predict move timing, enabling it to maintain meaningful clock dynamics throughout gameplay.

## Model Design
<img width="1354" alt="image" src="https://github.com/user-attachments/assets/feffcfec-188d-48d8-babe-35dae1e9b3a2" />

The model adopts a **Vision Transformer (ViT)** architecture with **8 layers**, **8 attention heads**, **256 embedding dimensions**, **256 hidden layer dimensions** in the feed-forward network (FFN), learned **absolute** positional encodings, and GELU nonlinearities. Within each transformer block, Leela’s smolgen mechanism (described below) also generates **relative** positional encodings for each attention operation. The first smolgen linear layer uses **16 dimensions**, while the second smolgen linear layer uses **128 dimensions**.

### Input Encoding
80 channels per square:

- **0-5**: P1 Pieces (K, Q, R, B, N, P)
- **6-11**: P2 Pieces (K, Q, R, B, N, P)
- **12**: EP Capture Square
- **13-14**: P1 Castling (Kingside, Queenside)
- **15-16**: P2 Castling (Kingside, Queenside)
- **17**: P1 Piece Mask
- **18**: P2 Piece Mask
- **19-23**: P1 Material Difference (Q, R, B, N, P)
- **24**: Checkers (squares occupied by checking P2 pieces)
- **25-29**: P1 Material Count (Q, R, B, N, P)
- **30**: 50-move rule (half-move clock)
- **31-39**: 4-move context, origin, and target squares
- **40**: Repetition (draw possibilities)
- **41-46**: Time Control (bins: 65, 125, 185, 305, 605)
- **47-50**: Increment (bins: 0.5, 1.5, 3.5)
- **51-57**: P1 ELO (bins: 1200, 1400, 1600, 1800, 2000, 2200)
- **58-64**: P2 ELO
- **65-72**: P1 Time (bins: 5, 10, 20, 30, 60, 120, 240)
- **73-79**: P2 Time

P1 always is the player to move. When P1 is black, the board mirrors to white's perspective (e.g., 2. e7e5 as 2. e2e4). Including explicitly encoded board features (e.g., material difference, repetition) improves network efficiency, especially in light-weight models like HC3. These were selected from [here](https://arxiv.org/pdf/2304.14918). 

### Positional Encoding
Both **absolute** and **relative** positional encodings are used. The relative positional encoding used comes from Leela's **smolgen**. In chess, positional relationships are position-dependent. For instance, a closed position with a locked center may require minimal long-range communication between bishops, while an endgame requires a lot.

Very roughly, two linear layers compress the square encodings into `sg_hidden2` dimensions. Then, biases for each attention head are generated by two more linear layers. More details can be found here: [Leela Transformer Progress](https://lczero.org/blog/2024/02/transformer-progress/).


### Output Heads
HC3 is trained on these tasks:

1. **Next Move Prediction**: Only geometrically possible moves were included and rounded to 2048.
2. **Outcome Prediction**
3. **Move Speed Prediction**: Enables human-like timing and proper use of time inputs while playing a human (bins: 0.5, 1.5, 2.5, 4.5, 9.5, 19.5, 59.5). Binning and converting this to a classification task helps avoid exploding gradients and allows the model to generalize across categories.
4. **Legal Moves**: Enhances global positional understanding and helps the model avoid tunnel-vision.
5. **Move Origin and Target Squares**: Improves learning of rarely-seen moves by tying them to the squares that make up those moves. Also provides the model with a granular, human-like understanding of the moves it plays. 

The output head uses a weighted average of token embeddings. The weights are learned parameters that are softmaxed before matmul. This provides the model with the flexibility to overlap similar tasks or distinguish dissimilar tasks as needed. All tasks are classification.

## Data Processing and Augmentation
Data is taken from the lichess database and each position is processed into a csv file with the following headers: 
`game_id, fen, repetition, max_time, increment, W_time, B_time, W_elo, B_elo, past_move1, past_move2, past_move3, past_move4, next_move, time_rem, outcome`

Since certain ELO categories, positions, and time controls are more common than others, certain categories are undersampled to maintain a healthy distribution. In addition, I follow Maia2 in also increasing the number of games played across ELO categories relative to games played within the same category. This allows the model to learn how higher ELO players play against lower ratings. 

### Before Augmentation 
This is what the data looks like before augmentation for the first 1M positions, encoding every position found in the lichess database. The 2D histogram is on a log scale with base e. I apologize in advance for its scuffness.

<img width="400" alt="image" src="https://github.com/user-attachments/assets/dddf1804-d6f0-442a-8a82-6b02f5304c0e" />
<img width="400" alt="image" src="https://github.com/user-attachments/assets/b98c5d02-44bb-40e2-a125-d38f446101a2" />
<img width="400" alt="image" src="https://github.com/user-attachments/assets/7ec05cd7-a47d-494f-91b1-738f08c62f1b" />
<img width="400" alt="image" src="https://github.com/user-attachments/assets/afb00916-c028-48fd-ae0f-044f11cb4f17" />
<img width="400" alt="image" src="https://github.com/user-attachments/assets/114a1528-0e4b-4b2f-86ca-24cec1b8c6db" />

### After Augmentation 
Augmentation is done by undersampling categories that appear too many times in the training data. The most dramatic undersampling at the game level comes from ignoring games played between players of the same ELO category 90% of the time and games played between one-off ELO categories 50% of the time. In addition, games with less than 30 plies and with a player under 1000 ELO are removed. Within a game, the rate of sampling is lowered the earlier the move until ply 15. This allows these positions to still be a few of the most frequent, but less dramatically so. This allows the model to "memorize" early, frequent positions like how a human would study theory.

<img width="400" alt="image" src="https://github.com/user-attachments/assets/07cdaf3d-755d-42b2-bf8c-aae9a327a8a3" />
<img width="400" alt="image" src="https://github.com/user-attachments/assets/90922cd5-d266-4432-8e74-c496f3cad2de" />
<img width="400" alt="image" src="https://github.com/user-attachments/assets/8625f67b-4c42-4b92-8a6e-00ceacf32a4f" />
<img width="400" alt="image" src="https://github.com/user-attachments/assets/7c0c4678-bcda-4709-8295-398b513223a1" />

## Model Training and Evaluation

### Training
The training dataset included 320M positions, or about 5M online games. The model was trained with a total batch size of 2048 for 1M iterations on a GH200 GPU from Lambda Labs. The maximum learning rate was set at 5e-4 with 1000 warmup iterations, and cosine decay to 5e-5. The total cost was about $110. 

### Evaluation
The final accuracies were achieved in the validation set: 
- **Next Move**: 54.4%
- **Move Speed**: 35.4%
- **Move Origin**: 66.9%
- **Move Target**: 57.6%
- **Game Outcome**: 68.0%

HC3 seems to outperform Maia1 and Maia2 on move-matching accuracy with 30x less training data than Maia2 and less training iterations. However, further testing is required, since the Maia dataset and HC3 dataset differs in many ways. For example, Maia's datasets exclude moves made in time trouble (under 30 seconds) and non-10 minute games. 

To understand HC3's accuracy better, I plotted each measure across various game settings. Here are my results:
![image](https://github.com/user-attachments/assets/3154d8fa-faf7-4e02-8b53-230ad9907471)
![image](https://github.com/user-attachments/assets/4c23b54a-a8ac-43af-9575-a9d7e5e065f0)
![image](https://github.com/user-attachments/assets/c7114c53-d13b-4b8b-a9b7-110e4ee812b1)
![image](https://github.com/user-attachments/assets/51b21903-1efb-4584-9be8-81d566115391)
![image](https://github.com/user-attachments/assets/99c61053-9ece-4417-bdc2-e4f26093aedc)
![image](https://github.com/user-attachments/assets/9e930615-b07f-4a61-b167-4c89debdf209)

Some interesting observations from this. Accuracy improves as the game progresses, likely because there is less material so the moves become more obvious. This is evident in the material difference and material count plots. Move-matching accuracy remains relatively constant throughout each time control until 1-minute games. This is not an intuitive conclusion, because you would expect higher time pressure in blitz formats to make moves more random. Additionally, move-matching accuracy remains constant even during time-pressure. However, a closer look needs to be taken, controlling for material count and other factors that may otherwise benefit move-matching accuracy.

Anecdotally, HC3 underperforms for its ELO class. Move timing is also somewhat strange and requires improvement, especially in slower formats. HC3 also has a tendency to randomly blunder, indicating that although it performs the classification task well, when it's wrong, it's wrong badly. For example, it often hangs a piece or mate in 1.  Below is an example of HC3 doing this in a game with Maia9.
<div align="center">
  <img width="300" alt="image" src="https://github.com/user-attachments/assets/adb66625-1075-4e12-ab61-96ad50d9e504" />
</div>

For those interested, HC3 is available to play with [here](https://lichess.org/@/HC2_18-20). The ELO is set at 1900.

## Next Steps
I've listed a few areas of improvement, along with possible solutions or next steps. 
- **Random Blunders**: Models like Lc0 benefit from using their evaluation output rather than their policy output to guide move choice. Further experimentation may be required to see if it improves move-matching accuracy, or at the very least maintains move-matching accuracy while reducing the number of obvious blunders (such as hanging mate in 1). A hybrid system is also possible, where the evaluation output is used when the model chooses to take more time.
- **Timing**: HC3's timing remains janky, especially in faster time controls where it is far too easy to flag. This likely requires more training iterations to resolve. Another solution can be to use perplexity of the policy output to influence timing, although in theory this should be learned by the model. I would love suggestions here. I also notice that in human games, the time you take on a move is influenced by the time your opponent took on the previous move. For example, you may respond to a 1 second blunder very quickly, but may also think for awhile if the opponent blunders after thinking for two minutes. 
- **Determinism**: Currently, the model chooses the most likely move, which is deterministic and can lead to repeated games. Further testing with different temperatures during random sampling is required to optimize the playing experience.

## Acknowledgements
This model was inspired by Maia's objective (https://www.maiachess.com/). The model is based on Lc0's ViT-like transformer architecture (https://lczero.org/blog/2024/02/transformer-progress/). Generic transformer model code is from NanoGPT (https://github.com/karpathy/nanoGPT). 