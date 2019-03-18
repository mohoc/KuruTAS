# Summary

## Memory Watch

A memory watch for the game can be found here:  
https://github.com/E-Sh4rk/KururinTAS/blob/master/Kuru%20Kuru%20Kururin%20(Europe).wch

Here are some explanations for important addresses:

|| Name           ||Address (IWRAM)|| Size   || Description                                                                                                             ||
|:---------------:| --------------- | ------- | ------------------------------------------------------------------------------------------------------------------------ |
| X / Y           | 0x4544 / 0x4548 | 32 bits | Position of the center of the helirin. If we only look at the 16 most significant bits, it gives the position in pixels. |
| XB / YB         | 0x454C / 0x4550 | 32 bits | Bump speed. A bump speed is applied when the helirin hits a wall, and then decreases until reaching.                     |
| XS / YS         | 0x4554 / 0x4558 | 32 bits | Input speed. It only depends on the direction pressed for the this frame.                                                |
| Angle           | 0x4572          | 16 bits | Angle of the helirin, 0 being vertical. `2^16` correspond to 360°, so for instance `2^16 / 360 * 90 = 16384` is 90°.     |
| Angle Rate      | 0x4574          | 16 bits | Rotation speed. It is 182 or -182 by default, but it momentary changes when the helirin hits a wall.                     |
| Default Rate    | 0x4576          | 16 bits | 182 when the helirin rotates clockwise, and -182 when it rotates counter-clockwise. Can change when touching a spring.   |
| Invulnerability | 0x4585          | 8 bits  | Number of invulnerability frames left. Grows to 20 when the helirin looses a heart, then decreases by 1 every frame until 0. This value is decremented before being used, so having it to 1 is equivalent to having it to 0. |
| MapW and MapH   | 0x313C / 0x313E | 16 bits | Size of the map in number of tiles. A tile is 8x8 so multiplying it by 8 gives the size of the map in pixel.             |
| Collision Mask  | 0x45D4          | 32 bits | Indicates which parts of the helirin are in collision with a wall.                                                       |

The collision mask must be read in binary. Each bit correspond to a point of the helirin.
If we number the bits from 0 (for the least significant) to 31, the different points are:

`16--14---12---10---8---6---4---2---(O)---1---3---5---7---9---11---13--15`

Note that the bits 17 to 31 are not used. The least significant bit correspond to the center of the helirin.
The helirin has a physical radius of 31px.

Note that collisions are checked after applying input speed and bump speed to the helirin, but an additional bump speed can still be applied after (if the helirin hits a wall).

## Physics and wall clips

The physics of the game is composed of a few simple mechanisms:

- A speed based on the input. It changes instantly depending on the input (no inertia, instant acceleration). If the helirin hits a wall after applying this speed, it is canceled (so the player has no control when the helirin hits a wall).
- A bump speed of 2px/frame, that is applied when the helirin hits a wall. It is applied in the direction of the vector starting from the collision point to the center of the helirin. It then decreases by 1/4 each frame until reaching 0.
- A constant rotation speed of (+/-)182. It can be temporarily changed to (+/-)1024 when hitting a wall, but it will then return to its initial value by steps of 91 per frame.
- An additional speed of 4px/frame, only when the center of the helirin hits the wall, in the opposite direction of the input. It shares the same variable as the bump speed. It was probably introduced in order to avoid wall clips, but actually it has the opposite effect: it can be abused in order to perform a wall clip. Moreover, it allows the helirin to move very fast when inside a wall.

For a more precise description, you can refer to the implementation of the KuruBot:  
https://github.com/E-Sh4rk/KururinTAS/blob/master/KuruBot/KuruBot/KuruBot/Physics.cs

These mechanisms can be abused in order to perform wall clips. The idea is the following:

1. We approach a wall, and we try to get a bump speed in the direction of that wall. We can't do that anywhere: there must be an angle in the wall order to get a bump speed in the right direction.
2. We let the bump speed push the center of helirin in the wall. The helirin must be parallel to the wall, otherwise an extremity will hit the wall before the center.
3. Just before the collision between the center and the wall, we must hold the opposite direction. In this way, at the next frame, an additional speed of 4px/frame will be applied in the direction of the wall.
4. We can then navigate in the wall by holding the opposite direction to that desired.

However, this is really hard to do by hand (even with an emulator): it requires a very precise combination of position and speed that is quite tricky to reach. This is why the KuruBot has been designed.

## Map and OOB

Wall clips allow us to get out of bounds. Although nothing is displayed graphically, there is walls and healing/ending zones in the OOB. More precisely:

- Objects like springs, bonuses or birds are not present in the OOB.
- The walls of the regular map are replicated in all directions. On some maps (those with a width that is not a power of two), walls of the left OOB can be shifted/cut, due to overflow of the x-position variable.
- Healing and ending zones are replicated in the right and bottom OOB. In the left OOB, zones are also replicated but they are slightly shifted 8px down and 7px left. They are not present at all in the top OOB.

For instance, here are the physical map of *Grassland 1* and *Machine Land 1* (images generated by KuruBot):

![Grassland 1][gl1]

![Machine Land 1][ml1]

This behavior may seem quite weird. Indeed, it can be explained by a very simple formula.
In order to compute collisions, the game has to determine, for each of the 17 physical points composing the helirin, what type of tile is behind.

The map is stored at the beginning of the EWRAM (0x02000000). The two first words contains the width and height of the map, and then (at address 0x02000004) is a matrix describing the map (as a sequence of 16-bits tile identifiers).

When the game wants to determine the tile behind the center of the helirin, it reads the 16-bits identifier at the following address:

`0x02000004 + 2*(y_tile % map_height)*map_width + 2*(x_tile % map_width)` with:

- `map_width` and `map_height` being the dimensions of the map. It is respectively available at address `0x02000000` and `0x02000002`.
- `x_tile` being the x-coordinate of the center of the helirin in number of tiles. A tile being 8x8, it is computed in this way: `x_tile = (X >> 16)/8`, with `X` being the x-coordinate of the helirin as stored in memory (see memwatch).
- Similarly, `y_tile = (Y >> 16)/8`.

The replication of the map in all directions is due to the use of the modulo operator. The differences observed between walls and healing/ending zones in the left and top OOB are explained by the following:

- When computing collisions with walls, the game consider the coordinates `X` and `Y` of the helirin as being unsigned. It results in an overflow at the left and at the top of the map. When the width of the map is not a power of 2, the x-coordinate overflow will change the result of the modulo: this explains the shift of the walls in *Machine Land 1*.
- When computing collisions with healing/ending zones, the game consider the coordinates `X` and `Y` of the helirin as being signed. Thus, there is no overflow at the left nor at the top, **BUT** `(y_tile % map_height)` being negative in the top OOB, the formula above gives an address outside of the matrix storing the tiles. It explains why there is no healing/ending zone in the top OOB. The slight shift of these zones in the left OOB can also be explained this way.

In order to visualize the physical map when being OOB in game, you can use the *OOB Viewer* script available here:  
https://github.com/E-Sh4rk/KururinTAS/tree/master/OoB%20Viewer

## KuruBot

We have designed a bot in order to help us to perform wall clips.
It is available here:  
https://github.com/E-Sh4rk/KururinTAS/tree/master/KuruBot (instructions & sources)  
https://github.com/E-Sh4rk/KururinTAS/releases (binaries)  

It is implemented in C#. The exact physics of the game has been replicated, and a custom A* algorithm is used in order to find shortest paths bewteen two states. Due to the complexity of the problem (there are more than 2^100 reachable states), it can't solve this problem exactly. In other words, there is no guarantee that the path found is optimal. The behavior and the level of approximation can be easily configured through some predefined configuration files:

- We can choose a precise search, or a less precise but faster one.
- We can choose a focused search or a more explorative one.
- We can focus on specific paths, such as damageless paths, paths without OOB, etc.

The bot is able to solve easy levels on its own (we just have to give it a relevant configuration).
For instance, in this TAS, the inputs for `Grassland 1` have been entirely computed by KuruBot.
For bigger levels, we have to plan a global itinerary and then use KuruBot on smaller segments.
Typically, solving a level using KuruBot looks like:

- **Me:** Please KuruBot, try to go from this position to the ending zone.
- **KuruBot:** *10 GB of RAM later...*
- **Me:** KuruBot, abort. Try to reach this little area instead. And please keep 2 hearts! It will be useful later.
- **KuruBot:** *3 sec later.* Pfff, too easy ;)
- **Me:** Now, please go from this new position to the ending zone by clipping in the wall.
- **KuruBot:** *3 sec later.* Easy peasy lemon squeezy!
- **Me:** Thank you, Kuru. You are so smart. I love you. I mean, really. We can do so much together!
- **KuruBot:** ...

[gl1]: img/map_gl1.png "Grassland 1"
[ml1]: img/map_ml1.png "Machine Land 1"