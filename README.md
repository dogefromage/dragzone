# dragzone

## Introduction
Dragzone is a __small, flexible and customizable__ library for mouse interactions.
The library relies on React.createPortal to place a fullscreen div onto the page whenever an target is being dragged. This allows shapes to be moved using the mouse even if the mouse strays outside the parent container. 
dragzone also features two hooks - useDraggable and useDroppable - which allow for customizable drag and drop functionality and data transfer over drag and drop.
To initialize the library, one must place the portal target at the end of the react tree.

## Object dragging - useMouseDrag

The ```useMouseDrag``` hook can be used to drag absolute positioned divs over the screen. The following example shows the a React component which can be moved by dragging it using the left mouse cursor.

```ts
// main.tsx

ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(
    <React.StrictMode>
        <App />
        /* place portal mount at end of document */
        <DragzonePortalMount />
    </React.StrictMode>,
)


// MyMovableBox.tsx

const MyMovableBox = ({ movingDisabled }: Props) => {
    const [ position, setPosition ] = useState(INIT_POS);

    /* A ref can be of use to store positions and values during a dragging action. */
    const moveRef = useRef<{
        startCursor: Point;
        lastCursor: Point;
    }>();

    const { handlers, catcher } = useMouseDrag({
        mouseButton: 0, // only start move with left cursor
        start: (e, cancel) => {
            if (movingDisabled) {
                cancel(); // a drag action can be canceled if necessary
                return;
            }

            console.log('Started moving my box.');

            moveRef.current = {
                startCursor: { x: e.clientX, y: e.clientY },
                lastCursor: { x: e.clientX, y: e.clientY },
            };
            e.stopPropagation();
        },
        move: e => {
            if (!moveRef.current) return;

            const delta = {
                x: e.clientX - moveRef.current.lastCursor.x,
                y: e.clientY - moveRef.current.lastCursor.y,
            };
            moveRef.current.lastCursor = { x: e.clientX, y: e.clientY };

            const nextPosition = moveMyBox(position, delta);
            setPosition(nextPosition);
        },
        end: e => {
            console.log('My box was moved.');
        }
    });

    return (<>
        /* The handlers should be added to the desired div. */
        <MyBox position={position} {...handlers} />
        /* The catcher will enable if a drag has been started. */
        {catcher}
    </>);
}

```

### Types
A config object can be optionally passed as second argument.

```ts
// parameters of useMouseDrag
useMouseDrag(
    interactions: [ /* a single interaction or an array of interaction can be passed */
        {
            /* if provided, only this mouse button will initialize an event */
            mouseButton?: number;
            /**
             *  start will get called after the mouseButton 
             *  has been pressed and the mouse has moved further
             *  than the specified deadzone.
             */
            start?: (e: React.MouseEvent, cancel: () => void) => void;
            /* move will get called per mouse move event if a drag has been started */
            move?: (e: React.MouseEvent) => void;
            /* end will be called at the end of a drag action. */
            end?: (e: React.MouseEvent) => void;
        }
    ],
    config: {
        /* Will set the css property cursor to provided value. */
        cursor?: string;
        /* Specifies number of pixels which the mouse can move whilst being pressed 
         * without starting a drag.
         * This can be used if a separate action requires a click event on the same div.
         */
        deadzone?: number;
    }
)

// return types of useMouseDrag
{
    /* Handlers to be placed on div */
    handlers: {
        onMouseDown: (e: React.MouseEvent) => void;
        onMouseMove: (e: React.MouseEvent) => void;
        onMouseUp: (e: React.MouseEvent) => void;
    };
    /* Can be placed anywhere in the app. Will automatically turn on or off. Uses React.createPortal. */
    catcher: JSX.Element | null;
}
```

## Drag and drop - useDraggable/useDroppable

Drag and drop consists of a draggable and a droppable element. The draggable element can be dragged using the mouse if the draggable attribute is specified (done automatically in hook). The draggable and droppable elements will fire a series of events depending on the current action. 
The ```useDraggable``` and ```useDroppable``` hooks will organize all events into a single hook. Furthermore, they will distinguish between different types of drag and drop events using a tag which must be shared between both targets. You are also able to pass data from the draggable to the droppable component. However, this data must be serializable using the ```JSON.stringify()``` function. This approach allows the transfer data also to be read inside the DragOver event which usually cannot be done without any side effects. Similar to raw drag and drop logic, default must be prevented inside the DragEnter, DragOver and DragLeave events such that an item can be dropped.
The ```useDroppable``` hook additionally kees a count of the number of enter and leaves to determine whether the cursor is still inside the droppable target. The resulting state is returned as ```isHovering```.

### Basic drag and drop setup:

```ts

// define common tag
const DND_TAG = 'drag_A_B';

const DraggableComponent = () => {
    const { handlers: dragHandlers } = useDraggable({
        tag: DND_TAG,
        start: e => {
            /* removes blurred dnd image */
            e.dataTransfer.setDragImage(new Image(), 0, 0);
        },
    });

    return (
        <div {...dragHandlers} />
    );
}

const DroppableComponent = ({ canDrop }) => {
    /**
     * Dropping can only be enabled after default is prevented.
     * It is not necessary to use the same handler three times
     * here it is practical.
     */
    const droppableHandler = (e: React.DragEvent) => {
        if (canDrop) {
            e.preventDefault();
        }
    }
    
    const { 
        handlers: dropHandlers, 
        //  Entering the droppable div will rerender the component and set isHovering; same for exit.
        isHovering,
    } = useDroppable({
        tag: DND_TAG,
        enter: droppableHandler,
        over: droppableHandler,
        leave: droppableHandler,
        drop: e => {
            alert('Dropped!');
            e.stopPropagation();
        },
    });

    return (
        <div {...dropHandlers} />
    );
}
```

### Drag and drop using custom transfer data:

```ts

// Custom transfer data object
// Must be JSON stringifyable
interface TransferData {
    id: string;
    color: string;
}

// in DraggableComponent:

// TransferData passed to generic useDraggable hook
const { handlers } = useDraggable<TransferData>({
    tag: DND_TAG,
    start: e => {
        // returning transfer data will bind it to the event
        return { id: '5', color: 'red' };
    },
});

// in DroppableComponent:

// same interface can be used for useDroppable
const { handlers } = useDroppable<TransferData>({
    tag: DND_TAG,
    enter: e => e.preventDefault(),
    over: e => e.preventDefault(),
    leave: e => e.preventDefault(),
    // every callback has transferData as second argument
    drop: (e, data) => {
        console.log(`Element ${data.id} has color ${data.color}.`);
        e.stopPropagation();
    },
});

```

### Types

```ts

function useDraggable<T extends {}>(props: {
    tag: string;
    start?: (e: React.DragEvent) => T | undefined | void;
    end?:   (e: React.DragEvent) => void;
}): {
    /* to be put on draggable element */
    handlers: {
        onDragStart: (e: React.DragEvent) => void;
        onDragEnd: (e: React.DragEvent) => void;
        draggable: boolean;
    };
};

function useDroppable<T extends {}>(props: {
    tag: string;
    enter?: (e: React.DragEvent, transfer: T) => void;
    over?:  (e: React.DragEvent, transfer: T) => void;
    leave?: (e: React.DragEvent, transfer: T) => void;
    drop?:  (e: React.DragEvent, transfer: T) => void;
}): {
    /* to be put on droppable element */
    handlers: {
        onDragEnter: (e: React.DragEvent) => void;
        onDragOver: (e: React.DragEvent) => void;
        onDragLeave: (e: React.DragEvent) => void;
        onDrop: (e: React.DragEvent) => void;
    };
    /* reactive state, can be used for rendering */
    isHovering: boolean;
};
```