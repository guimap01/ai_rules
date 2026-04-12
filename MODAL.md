# MODAL.md

Modal pattern rules for all AI agents working in this repository (`apps/client-app`, `apps/device-manager`).

---

## Rule — MANDATORY

Every modal must be implemented as a custom hook. No exceptions.

```ts
// ✅ Correct
export function useCreateFooModal(options) {
  const [isOpen, setIsOpen] = useState(false);
  // ... form, mutation logic ...
  const modal = <Dialog open={isOpen}>...</Dialog>;
  return { modal, open: () => setIsOpen(true), close: () => setIsOpen(false), isOpen };
}

// Call site
const foo = useCreateFooModal({ parentId });
return <>{foo.modal}<Button onClick={foo.open}>Create</Button></>;

// ❌ Wrong — modal state managed in the component
const [isOpen, setIsOpen] = useState(false);
return <><Button onClick={() => setIsOpen(true)}>Create</Button><Dialog open={isOpen}>...</Dialog></>;
```

## Rules

- `open()` takes no args for create modals. `open(entity)` for edit/delete modals.
- Always return `{ modal, open, close }`. Never return `setIsOpen`.
- Destructive confirmations must compose `useConfirmationModal` — never build a custom confirm layout.
- `onOpenChange` on edit modals must reset form + entity state on close.
- File: `src/components/modals/use<Action><Entity>Modal.tsx`, exported from `modals/index.ts`.
- Use the `/modal` slash command to generate any new modal.
