# React Components

## Basic Example

```tsx
type Props = {
  readonly title: string;
  readonly children?: React.ReactNode;
};

export const Card: React.FC<Props> = (props) => {
  const { children, title } = props;

  return (
    <section>
      <h2>{title}</h2>
      {children}
    </section>
  );
};
```

## Button with Event Handlers

```tsx
type Props = {
  readonly label: string;
  readonly onClick: () => void;
  readonly disabled?: boolean;
};

export const Button: React.FC<Props> = (props) => {
  const { onClick, disabled, label } = props;

  return (
    <button onClick={onClick} disabled={disabled}>
      {label}
    </button>
  );
};
```
