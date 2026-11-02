# ILikeTrains

**CTF:** GoogleCTF 2022  
**Category:** hardware  
**Original writeup:** <https://gist.github.com/szymex73/22940751792f3055cc7ccc769ab92d1b> — by **szymex73**. All credit goes to the original author; this is an archived copy.

---

ILIKETRAINS was a hardware challenge on Google CTF 2022 that involved a logic circuit built with a rail network in OpenTTD which had to be solved with the input bits being the solution.

For the challenge, we were given just the savegame file for the game.

In this writeup, I'll take a similar approach to what LiveOverflow did in his minetest challenge writeup for Google CTF 2019 tracing the logic circuit from the output to the inputs while trying to explain most of the steps I took along the way. The two challenges are very similar to each other (after the CTF concluded the organizers said that they've even used the same tool to generate the map, with some adaptations for a different game) but I'm sure this writeup will be helpful for someone anyway.

Because this writeup is pretty long since I tried to keep it detailed, here are some shortcuts to skip to relevant sections:

How do we even parse this map????

OpenTTD uses a custom savegame format, to get some clue on how it is parsed we can either read the OpenTTD source code (which is fairly big and hard to read imho) or we can scour the interwebz in search of projects that do some form of savegame parsing (for example OpenTTD_surveyor which parses the map and generates an image of it).

In the OpenTTD_surveyor source (specifically src/surveyor.py) we can see a function that tries to find specific chunks:

def set_map_bytes ( self ): """Populate the tiles from the parsed maps.""" maps_info = { b'MAPT' : [ 1 , 8 ], b'MAPH' : [ 1 , 8 ], b'MAPO' : [ 1 , 8 ], b'MAP2' : [ 2 , 8 ], b'M3LO' : [ 1 , 8 ], b'M3HI' : [ 1 , 8 ], b'MAP5' : [ 1 , 8 ], b'MAPE' : [ 1 , 8 ], b'MAP7' : [ 1 , 8 ], b'MAP8' : [ 2 , 8 ] } for map_name in maps_info : self . log_message ( f"Reading { map_name } ..." ) start = self . data . find ( map_name ) m = maps_info [ map_name ][ 0 ] n_tiles = len ( self . tiles ) offset = start + maps_info [ map_name ][ 1 ] tbs = self . data [ offset : offset + self . ncols * self . nrows * m ] [ self . tiles [ i ]. set_map_bytes ( map_name , tbs [ i * m :( i + 1 ) * m ]) for i in range ( n_tiles )] # [...]

It iterates over multiple map chunk names and sets information for a given tile taken from the given chunk data.

Reading the source further (src/tile.py) we can see that the MAPT chunk seems to contain information about what the given tile is, allowing us to distinguish between a rail track, grass bridges etc:

def parse_common ( self ): """Set the common parameters from the bits.""" self . zone = self . parse_map_bits ( b'MAPT' , 0 , 2 ) self . bridge = self . parse_map_bits ( b'MAPT' , 2 , 4 ) self . kind = self . parse_map_bits ( b'MAPT' , 4 , 8 ) self . height = self . parse_map_bits ( b'MAPH' , 0 , 8 ) self . owner = self . parse_map_bits ( b'MAPO' , 0 , 4 ) self . owner_tram = self . parse_map_bits ( b'M3LO' , 4 , 8 ) self . over_bridge_owner = None

We'll need to figure out how to determine the track direction though, for that I found it best to turn to the actual game source code where we can find very helpful information about what metadata is contained in which chunk for rail files (src/rail_cmd.cpp):

/* 4 * --------- * |\ /| * | \ 1/ | * | \ / | * | \ / | * 16| \ |32 * | / \2 | * | / \ | * | / \ | * |/ \| * --------- * 8 */ /* MAP2 byte: abcd???? => Signal On? Same coding as map3lo * MAP3LO byte: abcd???? => Signal Exists? * a and b are for diagonals, upper and left, * one for each direction. (ie a == NE->SW, b == * SW->NE, or v.v., I don't know. b and c are * similar for lower and right. * MAP2 byte: ????abcd => Type of ground. * MAP3LO byte: ????abcd => Type of rail. * MAP5: 00abcdef => rail * 01abcdef => rail w/ signals * 10uuuuuu => unused * 11uuuudd => rail depot */

Sidenote: I really recommend checking out Github Code Search for quickly and fairly precisely searching for specific code in a given repo without cloning it, grep.app is a good alternative if you don't want to use github or can't get beta access (github code search is still in beta at the time of writing). It's what I used for searching for relevant snippets of code in the OpenTTD repository and it helped a lot.

With this information, we are armed to start writing a map parser and determine information about the map tiles:

import lzma savefile = open ( './challenge.sav' , 'rb' ). read () data = lzma . decompress ( savefile [ 8 :]) # Map size taken from the game ncols = 4096 nrows = 4096 # Extract tile type into an array start = data . find ( b'MAPT' ) offset = start + 8 tbs = data [ offset : offset + ncols * nrows ] tiles = [ tbs [ i : i + 1 ] for i in range ( 4096 * 4069 )] # Extract tile metadata into an array (MAP5 contains rail direction meta) start = data . find ( b'MAP5' ) offset = start + 8 tbs = data [ offset : offset + ncols * nrows ] tile_meta = [ tbs [ i : i + 1 ] for i in range ( 4096 * 4069 )] # Helper functions to get a tile/meta at a given coordinate def get_tile_at ( x , y ): return tiles [ y * ncols + x ][ 0 ] def get_meta_at ( x , y ): return tile_meta [ y * ncols + x ][ 0 ] print ( get_tile_at ( 1 , 1 )) # 1,1 contains a grass tile

Baby steps along the track

With the above code we can probe around the map and retrieve the tile IDs for the tiles important to us:

GRASS = 0x00 # Grass/Empty tile RAIL = 0x10 # Normal rail track RAIL_UNDER = 0x14 # Rail track under a bridge RAIL_BRIDGE = 0x90 # Rail bridge endings

Split tracks have the same tile ID as normal straight rails so we will need to distinguish between them by checking the metadata of the rails.

For that we can yet again turn to the game source and see the bitfields they set in the header files for the tile segments (src/track_type.h) and we can write out our own global variables for those:

TRACK_BIT_X = 1 TRACK_BIT_Y = 1 << 1 TRACK_BIT_UPPER = 1 << 2 TRACK_BIT_LOWER = 1 << 3 TRACK_BIT_LEFT = 1 << 4 TRACK_BIT_RIGHT = 1 << 5 TRACK_BIT_CROSS = TRACK_BIT_X | TRACK_BIT_Y TRACK_BIT_HORZ = TRACK_BIT_UPPER | TRACK_BIT_LOWER TRACK_BIT_VERT = TRACK_BIT_LEFT | TRACK_BIT_RIGHT TRACK_BIT_3WAY_NE = TRACK_BIT_Y | TRACK_BIT_UPPER | TRACK_BIT_RIGHT TRACK_BIT_3WAY_SE = TRACK_BIT_X | TRACK_BIT_LOWER | TRACK_BIT_RIGHT TRACK_BIT_3WAY_SW = TRACK_BIT_Y | TRACK_BIT_LOWER | TRACK_BIT_LEFT TRACK_BIT_3WAY_NW = TRACK_BIT_X | TRACK_BIT_UPPER | TRACK_BIT_LEFT TRACK_BIT_ALL = TRACK_BIT_CROSS | TRACK_BIT_HORZ | TRACK_BIT_VERT INVALID_TRACK_BIT = 0xFF

With those, we can quickly perform checks if a given rail is a single one or if it's a split, useful for differentiating in which direction we should go down the line:

track_bits = [ TRACK_BIT_X , TRACK_BIT_Y , TRACK_BIT_UPPER , TRACK_BIT_LOWER , TRACK_BIT_LEFT , TRACK_BIT_RIGHT ] def is_single_track ( tile_info ): cnt = 0 for c in track_bits : if c & tile_info == c : cnt += 1 return cnt == 1

With this we can write a rudimentary trace function that given coordinates and a direction will try to follow that track until it reaches a split or encounters a dead end:

# Helper dict containing the coordinate differences for a given direction direction_diffs = { 'up' : ( - 1 , 0 ), 'down' : ( 1 , 0 ), 'left' : ( 0 , - 1 ), 'right' : ( 0 , 1 ) } # Helper function to determine the new direction based on the track we're currently on and the previous direction def get_new_direction ( tile , tile_meta , prev_direction ): # If we encounter a bridge, we'll follow it until its end if tile == RAIL_BRIDGE : return prev_direction # If we encounter a piece of straight rail, we keep the same direction elif tile_meta & TRACK_BIT_X == TRACK_BIT_X : return prev_direction elif tile_meta & TRACK_BIT_Y == TRACK_BIT_Y : return prev_direction # On turn track pieces, choose the correct one based on our previous direction elif tile_meta & TRACK_BIT_LOWER == TRACK_BIT_LOWER : if prev_direction == 'up' : return 'right' elif prev_direction == 'left' : return 'down' elif tile_meta & TRACK_BIT_UPPER == TRACK_BIT_UPPER : if prev_direction == 'right' : return 'up' elif prev_direction == 'down' : return 'left' elif tile_meta & TRACK_BIT_LEFT == TRACK_BIT_LEFT : if prev_direction == 'up' : return 'left' elif prev_direction == 'right' : return 'down' elif tile_meta & TRACK_BIT_RIGHT == TRACK_BIT_RIGHT : if prev_direction == 'left' : return 'up' elif prev_direction == 'down' : return 'right' def trace ( x , y , direction ): while True : print ( x , y , direction ) tile = get_tile_at ( x , y ) tile_meta = get_meta_at ( x , y ) # If the tile is a rail tile, follow it normally if tile in [ RAIL , RAIL_UNDER ]: # If we're on the single track, follow the direction based on the track under us if is_single_track ( tile_meta ): direction = get_new_direction ( tile , tile_meta , direction ) diff = direction_diffs [ direction ] x , y = x + diff [ 0 ], y + diff [ 1 ] continue # Else, we are on a split track with more than two directions available else : print ( 'Split!' ) break # Else if the tile is a bridge tile, we enter it and keep traversing in the same direction until we encounter # another bridge tile meaning we've reached the end of the bridge # We're doing this because tiles in between the bridge ends are not considered to be a part of the bridge # and can be any other tile elif tile == RAIL_BRIDGE : print ( 'Entering a bridge!' ) while True : # Traversing through tiles until we hit the other end of the bridge diff = direction_diffs [ direction ] x , y = x + diff [ 0 ], y + diff [ 1 ] tile = get_tile_at ( x , y ) if tile == RAIL_BRIDGE : print ( 'Hopping off of the bridge' ) # Finished crossing the bridge x , y = x + diff [ 0 ], y + diff [ 1 ] break # In other case if we encounter a grass tile we can assume we've reached a dead end elif tile == GRASS : print ( 'End!' ) return trace ( 27 , 31 , 'left' )

With this, we can successfully follow along a simple rail track without splits!

Spicing it up with some logic!

Now that we can follow a simple track we have to add functionality on what to do when we encounter a split track.

But before that, a word from our sponsor: LOGIC GATES!

In this challenge, the logic gates are implemented using the rails and signals. To first differentiate between the gates when we look at the map in the game, we can use this handy reference guide people in the OpenTTD community have made.

The key thing to note is that they all will be using a split track on their output (and remember, the approach here is tracing the circuit from the end), giving us an opportunity to fairly easily detect a gate and perform decisions before we follow the track further ignoring the gate.

Another great thing in this savegame is that all the gates keep the same orientation which means we don't have to check for any variations of a given gate.

With that out of the way we can now implement some code that will do the following when it encounters a split track:

First, see if the surrounding tiles match up with a known logic gate If so, perform actions on the logic gate (for now we'll take the first input as our new coordinate, later we'll plug it into a theorem prover)

Otherwise, decide which direction to go on the split track

We can cheat a little with the last point because going north will always lead us closer to the inputs, if it isn't possible to go north, we'll need to determine which way will lead us north (there will only be one such way).

First, we'll define ourselves layouts of the logic gates that will occur on our map:

# Define layouts of logic gates present not_gate = [ [ GRASS , RAIL , GRASS ], [ RAIL , RAIL , RAIL ], [ RAIL , RAIL , RAIL ], [ RAIL , RAIL , RAIL ] ] or_gate = [ [ RAIL , GRASS , GRASS , RAIL ], [ RAIL , RAIL , RAIL , RAIL ], [ RAIL , RAIL , RAIL , RAIL ], ] and_gate = [ [ RAIL , GRASS , RAIL ], [ RAIL , RAIL , RAIL ], [ GRASS , RAIL , RAIL ], ]

With those present, we can now write code to try and match them

# [...] # If the tile is a rail tile, follow it normally if tile in [ RAIL , RAIL_UNDER ]: # If we're on the single track, follow the direction based on the track under us if is_single_track ( tile_meta ): direction = get_new_direction ( tile , tile_meta , direction ) diff = direction_diffs [ direction ] x , y = x + diff [ 0 ], y + diff [ 1 ] continue # Else, we are on a split track with more than two directions available else : # Extract tiles around us that we can check against the NOT layout surroundings = [ [ get_tile_at ( x - 2 , y - 2 ), get_tile_at ( x - 2 , y - 1 ), get_tile_at ( x - 2 , y )], [ get_tile_at ( x - 1 , y - 2 ), get_tile_at ( x - 1 , y - 1 ), get_tile_at ( x - 1 , y )], [ get_tile_at ( x , y - 2 ), get_tile_at ( x , y - 1 ), get_tile_at ( x , y )], [ get_tile_at ( x + 1 , y - 2 ), get_tile_at ( x + 1 , y - 1 ), get_tile_at ( x + 1 , y )], ] if surroundings == not_gate : print ( f'Encountered a NOT gate at { x } , { y } !' ) # The input for this gate is at (x-2, y-1) relative to our position, for now let's set those coordinates # and re-set our heading as coming out of the input we'll be heading up direction = 'up' x , y = x - 2 , y - 1 continue # Extract tiles around us that we can check against the AND layout surroundings = [ [ get_tile_at ( x - 2 , y - 2 ), get_tile_at ( x - 2 , y - 1 ), get_tile_at ( x - 2 , y )], [ get_tile_at ( x - 1 , y - 2 ), get_tile_at ( x - 1 , y - 1 ), get_tile_at ( x - 1 , y )], [ get_tile_at ( x , y - 2 ), get_tile_at ( x , y - 1 ), get_tile_at ( x , y )], ] if surroundings == and_gate : print ( f'Encountered an AND gate at { x } , { y } !' ) # The input for this gate are (x-2, y-2) and (x-2, y), for now we'll use the leftmost one as our new coordinate direction = 'up' x , y = x - 2 , y - 2 continue # Extract tiles around us that we can check against the OR layout surroundings = [ [ get_tile_at ( x - 2 , y - 2 ), get_tile_at ( x - 2 , y - 1 ), get_tile_at ( x - 2 , y ), get_tile_at ( x - 2 , y + 1 )], [ get_tile_at ( x - 1 , y - 2 ), get_tile_at ( x - 1 , y - 1 ), get_tile_at ( x - 1 , y ), get_tile_at ( x - 1 , y + 1 )], [ get_tile_at ( x , y - 2 ), get_tile_at ( x , y - 1 ), get_tile_at ( x , y ), get_tile_at ( x , y + 1 )], ] if surroundings == or_gate : print ( f'Encountered an OR gate at { x } , { y } !' ) # The input for this gate are (x-2, y-2) and (x-2, y+1), again we'll use the leftmost one direction = 'up' x , y = x - 2 , y - 2 continue # If we have not matched any of the gates, we can see if the current split track # is a split without a way up north if tile_meta & TRACK_BIT_3WAY_SW == TRACK_BIT_3WAY_SW : # If we're going left or right on this track, we can go through normally if direction in [ 'left' , 'right' ]: diff = direction_diffs [ direction ] x , y = x + diff [ 0 ], y + diff [ 1 ] continue # Otherwise, we are heading up (heading down into here is not possible on this split) # so let's make a decision if we can head left or right else : if test_left_leads_north ( x , y ): direction = 'left' else : direction = 'right' diff = direction_diffs [ direction ] x , y = x + diff [ 0 ], y + diff [ 1 ] continue # Otherwise, it means that our split track has a way up that we can take else : direction = 'up' # Change the current heading diff = direction_diffs [ direction ] x , y = x + diff [ 0 ], y + diff [ 1 ] continue # [...]

You can see that this is a bit of code but it should be pretty easy to understand. There is one missing piece though, and it's the magical test_left_leads_north function that determines if heading left from a given location will lead to a track heading up.

We can check that by trying to follow the track left, and if we encounter a turn left (which will head down) we can return false early. Otherwise, we can try and follow until we encounter a split that has one of the tracks heading up at which point we can then return true:

Here is a sample implementation of that:

# Helper function that checks if we were to go left, we would arrive at a tile that lets us go "up" def test_left_leads_north ( x , y ): tile = get_tile_at ( x , y ) tile_info = get_meta_at ( x , y ) while tile in [ RAIL , RAIL_UNDER ]: # Check the tile "above" our current one neighbor_up = get_tile_at ( x - 1 , y ) # If the tile above is a valid rail tile, and the split we're on is either a 3way split, is a cross or has a right turn # we can go in this direction if ( neighbor_up in [ RAIL , RAIL_UNDER , RAIL_BRIDGE ]) and ( ( tile_info & TRACK_BIT_3WAY_NE == TRACK_BIT_3WAY_NE ) or ( tile_info & TRACK_BIT_X == TRACK_BIT_X ) or ( tile_info & TRACK_BIT_RIGHT == TRACK_BIT_RIGHT ) ): return True # If the tile we're on has a left turn but no other rails, that means it's the end of our way and we can't go up that way if ( tile_info & TRACK_BIT_LOWER == TRACK_BIT_LOWER ) and ( tile_info ^ TRACK_BIT_LOWER == 0 ): return False # In other case go one tile left and check again d = direction_diffs [ 'left' ] x , y = x + d [ 0 ], y + d [ 1 ] tile = get_tile_at ( x , y ) tile_info = get_meta_at ( x , y )

All aboard the Z3 train

Now that we can follow the track without any problems and we can detect logic gates, we can implement Z3 into our tracer.

But what is Z3 you might ask? Z3 is a theorem prover that can be used in cases like ours to determine the required input states (in our case the input tracks) to reach a given output state (the single output track).

But how do we plug it into the script? We will need to change our trace function a bit for that. Instead of returning only when we encounter the finish of a track, we will return on either the end of a track (which signifies an input) or when we encounter a gate.

When we encounter an input, we will return a variable that represents that input in a way that Z3 understands. But when we encounter a gate we will instead return the given gate's traced inputs wrapped in a logic gate representation in z3 ( z3.And , z3.Or , z3.Not ).

At the top of our script, we'll need to now import z3

from z3 import *

And now in the trace function, we'll do the following modifications

CACHED_GATES = {} # A dictionary that will store the result of gates we enter so we don't have to trace them again INPUTS = Bools ( ' ' . join ([ f'input_ { i } ' for i in range ( 32 )])) # List of our inputs def trace ( x , y , direction ): # [...] # Extract tiles around us that we can check against the NOT layout surroundings = [ [ get_tile_at ( x - 2 , y - 2 ), get_tile_at ( x - 2 , y - 1 ), get_tile_at ( x - 2 , y )], [ get_tile_at ( x - 1 , y - 2 ), get_tile_at ( x - 1 , y - 1 ), get_tile_at ( x - 1 , y )], [ get_tile_at ( x , y - 2 ), get_tile_at ( x , y - 1 ), get_tile_at ( x , y )], [ get_tile_at ( x + 1 , y - 2 ), get_tile_at ( x + 1 , y - 1 ), get_tile_at ( x + 1 , y )], ] if surroundings == not_gate : print ( f'Encountered a NOT gate at { x } , { y } !' ) # Check if we cached this gate already, if so return the cached version if f'NOT_ { x } - { y } ' in CACHED_GATES : return CACHED_GATES [ f'NOT_ { x } - { y } ' ] ret = simplify ( Not ( trace ( x - 2 , y - 1 , 'up' ))) CACHED_GATES [ f'NOT_ { x } - { y } ' ] = ret return ret # Extract tiles around us that we can check against the AND layout surroundings = [ [ get_tile_at ( x - 2 , y - 2 ), get_tile_at ( x - 2 , y - 1 ), get_tile_at ( x - 2 , y )], [ get_tile_at ( x - 1 , y - 2 ), get_tile_at ( x - 1 , y - 1 ), get_tile_at ( x - 1 , y )], [ get_tile_at ( x , y - 2 ), get_tile_at ( x , y - 1 ), get_tile_at ( x , y )], ] if surroundings == and_gate : print ( f'Encountered an AND gate at { x } , { y } !' ) # Check if we cached this gate already, if so return the cached version if f'AND_ { x } - { y } ' in CACHED_GATES : return CACHED_GATES [ f'AND_ { x } - { y } ' ] ret = simplify ( And ( trace ( x - 2 , y - 2 , 'up' ), trace ( x - 2 , y , 'up' ))) CACHED_GATES [ f'AND_ { x } - { y } ' ] = ret return ret # Extract tiles around us that we can check against the OR layout surroundings = [ [ get_tile_at ( x - 2 , y - 2 ), get_tile_at ( x - 2 , y - 1 ), get_tile_at ( x - 2 , y ), get_tile_at ( x - 2 , y + 1 )], [ get_tile_at ( x - 1 , y - 2 ), get_tile_at ( x - 1 , y - 1 ), get_tile_at ( x - 1 , y ), get_tile_at ( x - 1 , y + 1 )], [ get_tile_at ( x , y - 2 ), get_tile_at ( x , y - 1 ), get_tile_at ( x , y ), get_tile_at ( x , y + 1 )], ] if surroundings == or_gate : print ( f'Encountered an OR gate at { x } , { y } !' ) # Check if we cached this gate already, if so return the cached version if f'OR_ { x } - { y } ' in CACHED_GATES : return CACHED_GATES [ f'OR_ { x } - { y } ' ] ret = simplify ( Or ( trace ( x - 2 , y - 2 , 'up' ), trace ( x - 2 , y + 1 , 'up' ))) CACHED_GATES [ f'OR_ { x } - { y } ' ] = ret return ret # [...] # [...] elif tile == GRASS : return INPUTS [ y - 5 ] # We're doing y - 5 because the inputs start at y=5, this way we can quickly grab an index

You may notice simplify() calls around our gate functions. The purpose of those is to simplify the logic circuit that was returned to remove any redundant computation.

Now that our trace function returns a big logic circuit, we can utilize the Z3 solver to find a solution for it and extract the inputs.

# Position of the last rail tile in the circuit start_x = 3740 start_y = 7 # Generate the circuit circuit = trace ( start_x , start_y , 'up' ) # Instantiate a z3 Solver s = Solver () # Add a constraint to it, that says we want the output of our circuit to be true s . add ( circuit == True ) # We check if the model is solvable, this should print "sat" print ( s . check ()) # We generate a solution model containing our solution bits model = s . model () # We then extract the bits one by one from the model by referencing the inputs we generated before output = [ bool ( model [ INPUTS [ i ]]) for i in range ( 32 )] # Then, we assemble and print the flag flag = 'CTF{' + '' . join ([ str ( int ( a )) for a in output ]) + '}' print ( flag )

And with that, we get the flag!
