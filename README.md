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

TODO