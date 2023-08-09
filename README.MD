# Items Picker

After unpacking the ItemsPicker.zip archive and opening the ItemsPicker.uproject file, you need to load all the objects in the game field:

<img src="./MarkdownPics/LoadObjects.png" alt="" data-canonical-src="" width="500" height="" />

After that you should see all the objects.

## 1. Enable keyboard mapping.

Created mapping for using the keyboard is connected at the beginning of the game.
This mapping describes how to use:

- _TAB_ key (a link to enhanced input that implements a change in the display of widgets);
- _Q_ key (link to enhanced input that implements picking up, placing in the inventory and removing from the playing field of the object);
- _E_ key (link to enhanced input that implements the removal of one position of the object from the inventory, and throwing this Item into the game field);
- _Esc_ key (link to enhanced input that implements the exit from the game).

At the same time, the old approach regarding 'Engine - Input': 'Axis and Action mappings' is ignored.
This is a simple assignment of the corresponding key in the project settings with an indication of what it is responsible for. In Unreal Engine 5, it is not desirable to do that, so I don't do it that way.

## 2. Create widgets.

Three widgets are created:

Inventory Widget - 'BPW_Inventory'\
Instructions Widget - 'BPW_Instructions'\
Inventory Stack Icon - 'BPW_PickUpItem'

- 'BPW_Inventory'. In addition to 4 text labels and a border with rounded corners, there is a 'WrapperBox', within which new widgets of objects stored in the inventory will be created. 'InventoryBox' is marked as IsVariable - so that it can be accessed in blueprints.

- 'BPW_Instructions'. A lighter copy of the previous widget - missing the 'WrapperBox' and the 'Inventory' label. This widget only gives a hint how to use the main keys.

- 'BPW_PickUpItem'. This is an 'overlay' that sits on top of the parent widget. A button has been added (you can enable the function of turning on the mouse cursor at the beginning of the game in order to select a specific item in the inventory by pressing the cursor, or develop a system for switching between positions with keyboard arrows in the future). Added icon - it is drawn with a picture attached to the corresponding blueprint of the rising object (this process is carried out in the 'AddNewStack' function in 'BP_Player'). Text with the number of objects in the stack. It is important to note that there are inventory sizes (this is the number of stacks that can be placed in the inventory - by default it is: 8 in 'BP_Player') and stack size (this is the maximum number of objects of the same type that can be placed "one on top of another" - for objects object 'BP_Ball' this parameter by default: 3). At the beginning of the game, the text is not visible, as soon as the second object appears in the stack, this text will light up, as soon as it decreases to 1, it will disappear again. The ItemIcon, ItemStackSizeText elements are created with the IsVariable mark - in order to edit them in the future (the same can be done with the ItemButton, in order to implement the selection of objects in the stack in the future when clicking with the mouse).

  Variables are created here that store ItemClass, MaxStack - they are filled during creation, as well as CurrentStack - updated in case of adding or removing objects.

At the 'PickingTesting' level - there are 25 instances of BP_Ball in the game field. Accordingly, you can place 8x3 balls in the inventory (8 stacks with 3 balls each) and only one ball will remain, which will not be added until there is room in the stack.

## 3. Draw widgets.

The 'Begin Play' event creates widgets for the inventory and game instructions.

<img src="./MarkdownPics/Instruction_StartUp.png" alt="" data-canonical-src="" width="450" height="450" />
<img src="./MarkdownPics/Inventory_StartUp.png" alt="" data-canonical-src="" width="450" height="450" />

But which widget will be displayed depends on the TAB key.
At the beginning of the game, the inventory is hidden and we see instructions on how to pick up and throw an item, and how to show the inventory.

When you press the TAB key, another widget will be shown - the inventory (which also has instructions), the smaller widget will be hidden.

The widget with the icon of the corresponding object for the inventory obviously does not need to be drawn at the beginning of the game. It will be drawn in the 'AddNewStack' function - which is called if we see an object nearby that can be lifted, there is no place for it in any stack (or there is no stack at all) and there is space in the inventory.

## 4. Add the required variables to the player's blueprint.

The player is 'BP_Player'.

When creating widgets, choose promote to variable - this is how to create InventoryWidget and InstructionsWidget variables. Boolean variable InventoryShowing - will help organize switching between them when pressing TAB. The InventoryCapacity variable is created in the player - how many stacks can be created in the inventory (by default 8). Next, you will need to read the size of the array with children in InventoryBox and make sure that this parameter does not exceed InventoryCapacity. When searching for objects available for lifting, they will be stored in the AvailableItems array, among which NearestItem will be found.

## 5. Create blueprints of the floor and objects.

Game Object - 'BP_PickableParent'\
Successor to the previous blue print, pickable ball - 'BP_Ball'\
Floor - 'BP_Floor'

It is desirable to have sufficiently adequate objects: such that you can suddenly kick with your foot and they will roll to the side, so for this they should have 'SimulatePhysics' enabled in 'StaticMesh'. So that objects do not fall under the floor at the beginning of the game, you need to create a blueprint of the floor with 'CollisionBox'. In the 'CollisionBox' settings, choose the 'PhysicsBody' reset in the column of collision resets - so that the balls do not fall through the floor.

Next, create a parent blueprint for the objects that we pick up. In the inherited blueprints, the icon (for the inventory) and the maximum number of objects in the stack will differ.

It has: an empty 'StaticMesh' - the successor will overwrite this component with its non-empty mesh, 'CollisionSphere' - makes it possible to "see" this item when searching via the 'GetAllOverlappedActors' function.

It is important to note here the good and bad design of the placement of these two components in the tree.

### Bad design

<img src="./MarkdownPics/BP_PickableParent_Root_BadDesign.png" alt="" data-canonical-src="" width="500" height="" />

This design with 'StaticMesh' under or above 'CollisionSphere' (which makes no difference) is the first thing you get after adding the appropriate components. With this design, when calling the 'GetActorLocation' function for the corresponding ball, you will get the coordinates of the ball with its position after creation (at the start of the game or after 'SpawnActor'). And if you suddenly move it, the 'BP_Player' player will stop "seeing" with it (when calling the 'GetAllOverlappedActors' 'CollisionSphere' function, the player will be far from the ball's initial coordinates).

### Good design

<img src="./MarkdownPics/BP_PickableParent_Root_GoodDesign.png" alt="" data-canonical-src="" width="500" height="" />

You need to drag 'StaticMesh' to the DefaultSceneRoot position and take it for yourself.
Now 'CollisionSphere' belongs to the mesh and gives its current coordinates, not those after initialization.

In the mesh settings > Collision, you need to select the 'WorldDynamic' preset and mark the 'PhysicsBody' and 'WorldDynamic' to block - so as not to intersect with the moving player's body and not sink under the floor.

It is 'CollisionSphere' that is responsible for finding the object and transferring its coordinates - that's why here for all objects choose Overlap.

Create variables in the parent blueprint: 'ItemClass' - in the inherited blueprint specify which class it represents, 'ItemIcon' - in the inherited blueprint choose an icon image for the inventory, 'StackSize' - also specified in the inherited blueprint.

## 6. Create a blueprint interfaces for communication between player, game objects, and inventory stack icon blueprints.

After pressing the Q key, an OpenInventory signal will be generated - which will receive the corresponding nearest object. It must make sure that it is addressed by the player, and not by another game object, and after that the game object packages the presentable information and sends it with the GetItemData signal.

Event GetItemData: The player should get the presentable information (class, icon and stack size) from the game object.

At the same time, the player sends signals: TouchItemData - read the current state of the icon (class, current number of objects in the stack, maximum number). And if there is a place for the new object in the existing stack, the object will be added there by the AddItemToStack signal (which the icon will receive).

If there is no room in the existing stack, the inventory will check to see if it is full, and if there is room for a new stack, it will be created. In such case, the object in the playing field will be destroyed.

Event DropItem: Respond to the signal about dropping the object from the inventory (the signal comes from the icon) - create a new object (with the appropriate coordinates and rotation) and send a signal to the game object PushItemAway: where the parent class has already written how it will be impulse is added to the object.

After pressing the E key, access to the last icon in the stack will be obtained (in the future, it will be possible to organize the selection of icons with keyboard keys or the mouse) and send a signal to the TakeFromInventory stack icon.
Event TakeFromInventory: a signal will be generated to reset the object of the corresponding DropItem class. In the same event, the counter will be reduced and updated on the icon, or the icon itself will be destroyed if it was the last object.

Accordingly, you can distribute signals on different blueprints:

BPI_Keyboard - OpenInventory and TakeFromInventory are generated directly after pressing the corresponding keys on the keyboard, received in 'BP_Player' and passed on.

BPI_Picking - GetItemData, DropItem, TouchItemData, AddItemToStack and PushItemAway are direct data exchanges between blueprints.

## 7. Organize the game process.

**Event graph in BP_Player:**

It is necessary to investigate so that during the game all signals and events have an adequate sequence, it is also desirable at the right moment to take away from the player the opportunity to control the game while the playing field is processed with the search for a new object, to protect against premature resetting, and other things.

_Event Enhanced IA_InventoryShowHide:_

- The first press - the inventory is shown, on the next it is hidden, on the next it is shown again and so on.

_Event Enhanced IA_PickUpStuff:_

- The Q key is pressed and the player's ability to control the game is turned off.
- The player stops.
- TouchAllAvailableItems: read all OverlappedActors and add the unique descendants of the BP_PickableParent class to an empty array. If the array remains empty after the analysis, give controls back to user. So it will be possible to wait until the player presses the Q key again.
- GetNearestItem: note that all objects must be within the 'CollisionSphere', so its radius can be used when searching for the nearest object. For each object in the array, you can calculate the distance to the player, update this distance if it is less than the previous one, and overwrite the variable with a reference to the nearest object.
- If no adequate nearby object was found (because we screened out all non-game objects that could get into the array) give controls back to user.
- If found: we send the OpenInventory signal to the parent class of the object, which packs information about this object for us.

_Event GetItemData:_

- First, check whether the corresponding object is programmed for stacking. We can create such objects that will occupy an entire stack (respectively, their stack size is equal to one).
- If the object is stacked, then check whether there is room for it in the existing stacks. Here we need the InventoryWidget variable, which will give us access to the InventoryBox variable from which we will take an array of existing stacks to each of which we will send a TouchItemData signal.

- We compare the class of the test class with the class in the widget, if there is a match - we check if there is room for it in the stack, if there is room - we send the AddItemToStack signal.

<img src="./MarkdownPics/TouchItemData_implementation.png" alt="" data-canonical-src="" width="500" height="" />

In the BPW_PickUpItem blueprint, you need to implement the TouchItemData signal - by connecting all these icons.

_Event AddItemToStack (BPW_PickUpItem):_

The stack counter is incremented, the text label is updated, and text visibility is set as necessary.

Back to **Event graph in BP_Player:**

- If a place is found - destroy the object in the playing field. Give control to the player.

- If there is no room in the stack, or if the object does not stack - you need to check if there is room in the inventory for a new stack. If there is, a new stack is created, the object in the playing field is destroyed and control is returned to the player.

- If there is no room in the inventory for a new stack, then the object is **not destroyed** and control is returned to the player.

_Event Enhanced IA_ThrowStuff:_

- After pressing the E key, we take the last widget in the widget array from the InventoryBox. If such a widget is found, we generate the TakeFromInventory signal.
  If there is none, then the game does not respond in any way - the inventory is empty.

_Event TakeFromInventory (BPW_PickUpItem):_

- Information about the class that will be released from the inventory is packed and a DropItem signal is generated for BP_Player.

- The stack is decremented. If it is equal to zero after that, the widget is destroyed.

- If there is at least one object left in the stack - you need to check whether we show the text label (if there is only one object in the stack, we do not show the label). We also update the quantity indicated in the text label.

Back to **Event graph in BP_Player:**

_Event DropItem:_

- An object with the appropriate position and rotation is generated in the field.

- The PushItemAway signal is generated for the BP_PickableParent object's parent class.

_Event PushItemAway (BP_PickableParent):_

- The initial impulse for the newly created object is calculated. This will add realism when dropping the object.

_Event Enhanced IA_Quit:_

- When the user presses Escape, the desktop window with the game will be closed (this functionality is designed to export the game to PC: Windows, Linux).

## You can enjoy the game.

<img src="./MarkdownPics/GamePlay_showcase.gif" alt="" data-canonical-src="" width="" height="" />
