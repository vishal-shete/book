# Stepper Based Tally Sync Approch

## Overview

This document outlines the architecture for implementing a stepper-based workflow system where a parent stepper machine orchestrates multiple child step machines through event-driven communication and automatic progression via `onDone` events.

## Architecture Principles



### 1. **Single Parent Machine (Stepper)**

## UI Layout Diagram


## UI Layout Diagram

```text
┌───────────────────────────────────────────────┐
│ Stepper │               Slot                  │
│         │                                     │
│         │                                     │
│         │                                     │
│         │                                     │
│         │─────────────────────────────────────│
│         │      Cancel     |     Next           │
└─────────┴─────────────────────────────────────┘



- Controls overall workflow progression
- Manages step transitions
- Handles NEXT button state
- Accumulates data from all steps

### 2. **Multiple Child Machines (Steps)**

- Each step is an independent machine
- Handles step-specific business logic
- Communicates with parent via events
- Sends completion data via `onDone`

### 3. **Single Slot System**

- One content slot reused for all step UIs
- Content changes based on current step
- No multiple slots or complex routing

### 4. **Event-Driven Communication**

- Child → Parent: `NEXT_ENABLE`/`NEXT_DISABLE` events
- Parent → Child: `NEXT_STEP` event
- Child → Parent: `onDone` (automatic)

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Stepper Machine (Parent)                     │
├─────────────────────────────────────────────────────────────────┤
│  States:                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │   step1     │  │   step2     │  │   step3     │            │
│  │             │  │             │  │             │            │
│  │ • Invoke    │  │ • Invoke    │  │ • Invoke    │            │
│  │   child     │  │   child     │  │   child     │            │
│  │ • Listen    │  │ • Listen    │  │ • Listen    │            │
│  │   for       │  │   for       │  │   for       │            │
│  │   events    │  │   events    │  │   events    │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│         │              │              │                        │
│         │              │              │                        │
│         ▼              ▼              ▼                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Single Content Slot                            │ │
│  │  [Step1 UI] → [Step2 UI] → [Step3 UI]                     │ │
│  │  (Same slot, different content)                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Workflow Steps

### 1. **Step Initialization**

```
1. Stepper starts in "step1" state
2. Stepper invokes FetchDataTree in content slot
3. FetchDataTree shows its UI
4. FetchDataTree sends NEXT_ENABLE event to parent
5. Parent enables NEXT button
```

### 2. **Step Completion**

```
6. User clicks NEXT button
7. Parent sends NEXT_STEP event to FetchDataTree
8. FetchDataTree goes to final state
9. FetchDataTree sends onDone with data to parent
10. Parent receives onDone and transitions to "step2"
```

### 3. **Step Progression**

```
11. Parent updates context: currentStep = 2
12. Stepper UI automatically re-renders
13. Parent invokes TallyConfigTree in same slot
14. New step UI appears
15. Process repeats for each step
```

## Data Flow

### **Child Machine Data Preparation**

```typescript
final: {
  type: "final", // Triggers onDone in parent
  data: ({ context }) => ({
    SRData: context.SRData,
    warehouseData: context.warehouseData,
    TotalSales: context.TotalSales,
    metadata: {
      completedAt: new Date().toISOString(),
      step: "fetchData",
      status: "success"
    }
  })
}
```

### **Parent Machine Data Storage**

```typescript
onDone: {
  target: "step2",
  actions: assign(({ event, context }) => ({
    currentStep: 2,
    stepData: {
      ...context.stepData,
      step1: {
        ...event.output, // Data from child
        receivedAt: new Date().toISOString()
      }
    }
  }))
}
```

### **Data Access in Subsequent Steps**

```typescript
// Child machine gets parent context via input
context: ({ input }) => ({
  ...input, // Contains all previous step data
  localData: {}
}),

// Access previous step data
entry: ({ context }) => {
  const step1Data = context.stepData.step1;
  console.log("Previous step data:", step1Data);
}
```

## Event Communication

### **Child → Parent Events**

- **`NEXT_ENABLE`**: Enables NEXT button when step is ready
- **`NEXT_DISABLE`**: Disables NEXT button when step is not ready

### **Parent → Child Events**

- **`NEXT_STEP`**: Signals child to complete and go to final state

### **Automatic Events**

- **`onDone`**: Automatically sent when child reaches final state

## Implementation Structure

### **Parent Machine (Stepper)**

```typescript
export const StepperMachine = createMachine({
	id: "STEPPER_MAIN",
	initial: "step1",

	context: {
		currentStep: 1,
		stepData: {},
		nextButtonEnabled: false,
	},

	states: {
		step1: {
			invoke: [
				{
					src: "FetchDataTree",
					id: "CONTENT_SLOT",
					onDone: {
						target: "step2",
						actions: assign(({ event }) => ({
							currentStep: 2,
							stepData: { ...context.stepData, step1: event.output },
						})),
					},
				},
			],

			on: {
				NEXT_ENABLE: {
					actions: assign(() => ({ nextButtonEnabled: true })),
				},
				NEXT_DISABLE: {
					actions: assign(() => ({ nextButtonEnabled: false })),
				},
			},
		},
	},
});
```

### **Child Machine (Step)**

```typescript
export const StepMachine = createMachine({
	id: "STEP",
	initial: "idle",

	states: {
		idle: {
			entry: "loadData",
			on: { DATA_LOADED: "ready" },
		},

		ready: {
			entry: [sendTo("PARENT", { type: "NEXT_ENABLE" })],

			on: {
				NEXT_STEP: {
					target: "final",
				},
			},
		},

		final: {
			type: "final",
			data: ({ context }) => context.result,
		},
	},
});
```
