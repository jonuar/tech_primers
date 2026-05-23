# PyTorch Cheatsheet

## Mental Model

PyTorch is built around **tensors** (n-dimensional arrays with GPU support) and **autograd** (automatic differentiation). The training loop is always: forward pass → compute loss → backward pass → update weights. You define models as classes inheriting from `nn.Module`. Everything explicit — PyTorch doesn't hide what's happening, which makes debugging tractable.

---

## Install & Minimal Setup

```bash
# CPU only
pip install torch torchvision

# CUDA (check your CUDA version first: nvcc --version)
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121

# Verify
python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
```

---

## Core Concepts

### 1. Tensors

```python
import torch

# Create tensors
x = torch.tensor([1.0, 2.0, 3.0])          # from list
x = torch.zeros(3, 4)                        # all zeros, shape (3,4)
x = torch.ones(3, 4)
x = torch.rand(3, 4)                         # uniform [0, 1)
x = torch.randn(3, 4)                        # normal N(0,1)
x = torch.arange(0, 10, step=2)             # [0, 2, 4, 6, 8]

# Shape / type inspection
x.shape          # torch.Size([3, 4])
x.dtype          # torch.float32
x.device         # device(type='cpu')

# Shape manipulation
x.reshape(2, 6)
x.view(2, 6)     # view shares memory — reshape copies if needed
x.unsqueeze(0)   # add dim: (3,4) → (1,3,4)
x.squeeze()      # remove size-1 dims
x.permute(1, 0)  # transpose: (3,4) → (4,3)

# Indexing — same as NumPy
x[0]             # first row
x[:, 1]          # second column
x[x > 0.5]      # boolean mask
```

### 2. Device Management

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# Move tensors and models to device
x = x.to(device)
model = model.to(device)

# .cuda() / .cpu() are shortcuts
x = x.cuda()
x = x.cpu()
```

### 3. Autograd

```python
# requires_grad=True tells PyTorch to track operations for backprop
x = torch.tensor([2.0], requires_grad=True)
y = x ** 2 + 3 * x       # y = 4 + 6 = 10
y.backward()               # compute dy/dx
print(x.grad)              # tensor([7.]) — dy/dx = 2x + 3 = 7

# Stop gradient tracking (inference, eval mode)
with torch.no_grad():
    output = model(input)

# Detach a tensor from the graph
value = x.detach().numpy()
```

### 4. Defining a Model

```python
import torch.nn as nn

class MLP(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim):
        super().__init__()
        # Define layers in __init__
        self.layers = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(hidden_dim, output_dim)
        )

    def forward(self, x):
        # Define the forward pass
        return self.layers(x)

model = MLP(input_dim=128, hidden_dim=256, output_dim=10).to(device)
print(model)
```

### 5. Loss Functions & Optimizers

```python
# Common loss functions
criterion = nn.CrossEntropyLoss()     # multi-class classification (includes softmax)
criterion = nn.BCEWithLogitsLoss()    # binary classification (includes sigmoid)
criterion = nn.MSELoss()              # regression

# Optimizers
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3, weight_decay=1e-4)
optimizer = torch.optim.SGD(model.parameters(), lr=0.01, momentum=0.9)

# Learning rate scheduler
scheduler = torch.optim.lr_scheduler.StepLR(optimizer, step_size=10, gamma=0.1)
```

### 6. DataLoader

```python
from torch.utils.data import Dataset, DataLoader

class MyDataset(Dataset):
    def __init__(self, X, y):
        self.X = torch.tensor(X, dtype=torch.float32)
        self.y = torch.tensor(y, dtype=torch.long)

    def __len__(self):
        return len(self.X)

    def __getitem__(self, idx):
        return self.X[idx], self.y[idx]

dataset = MyDataset(X_train, y_train)
loader = DataLoader(dataset, batch_size=32, shuffle=True, num_workers=4)
```

### 7. Training Loop

```python
def train_epoch(model, loader, criterion, optimizer, device):
    model.train()  # activates dropout, batch norm training behavior

    total_loss = 0.0
    for X_batch, y_batch in loader:
        X_batch, y_batch = X_batch.to(device), y_batch.to(device)

        optimizer.zero_grad()           # clear gradients from previous step
        outputs = model(X_batch)        # forward pass
        loss = criterion(outputs, y_batch)
        loss.backward()                 # backprop
        optimizer.step()                # update weights

        total_loss += loss.item()

    return total_loss / len(loader)
```

### 8. Evaluation Loop

```python
def evaluate(model, loader, criterion, device):
    model.eval()  # deactivates dropout, uses running stats in batch norm

    total_loss, correct = 0.0, 0
    with torch.no_grad():  # no gradient computation needed
        for X_batch, y_batch in loader:
            X_batch, y_batch = X_batch.to(device), y_batch.to(device)
            outputs = model(X_batch)
            loss = criterion(outputs, y_batch)
            total_loss += loss.item()
            preds = outputs.argmax(dim=1)
            correct += (preds == y_batch).sum().item()

    accuracy = correct / len(loader.dataset)
    return total_loss / len(loader), accuracy
```

### 9. Save & Load

```python
# Save only weights (recommended)
torch.save(model.state_dict(), "model.pt")

# Load weights
model = MLP(128, 256, 10)
model.load_state_dict(torch.load("model.pt", map_location=device))
model.eval()

# Save full model (less portable — tied to class definition)
torch.save(model, "full_model.pt")
model = torch.load("full_model.pt")
```

---

## Most-Used Patterns

### Freeze Layers (Transfer Learning)

```python
# Freeze all layers
for param in model.parameters():
    param.requires_grad = False

# Unfreeze only the final classifier
for param in model.classifier.parameters():
    param.requires_grad = True
```

### Gradient Clipping (prevents exploding gradients in RNNs/Transformers)

```python
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
# Call this AFTER loss.backward() and BEFORE optimizer.step()
```

### Inference Pipeline

```python
model.eval()
with torch.no_grad():
    input_tensor = torch.tensor(raw_input, dtype=torch.float32).unsqueeze(0).to(device)
    output = model(input_tensor)
    prediction = output.argmax(dim=1).item()
```

---

## Gotchas

- **Forgetting `optimizer.zero_grad()`** — gradients accumulate by default. Call it at the start of every training step.
- **Forgetting `model.eval()` during inference** — dropout stays active, batch norm uses batch stats instead of running stats → inconsistent predictions.
- **Tensor on wrong device** — inputs and model must be on the same device. Always `.to(device)` both.
- **Loss is `nan` early in training** — often a learning rate that's too high, or log(0) from missing epsilon. Check input normalization.
- **`view` vs `reshape`** — `view` requires contiguous memory; if the tensor isn't contiguous (after `permute`, `transpose`), call `.contiguous()` first or use `reshape`.
- **`CrossEntropyLoss` already includes softmax** — don't add `nn.Softmax()` in your model if using it or you'll double-apply.
- **Data type mismatch** — classification labels must be `torch.long` (`int64`), not `torch.float32`.

---

## Quick Links

- [PyTorch Docs](https://pytorch.org/docs/stable/)
- [PyTorch Tutorials](https://pytorch.org/tutorials/) — the official ones are genuinely good
- [nn.Module reference](https://pytorch.org/docs/stable/generated/torch.nn.Module.html)
- [torchvision models](https://pytorch.org/vision/stable/models.html) — pretrained models for transfer learning
- [Weights & Biases](https://wandb.ai) — experiment tracking that pairs well with PyTorch
