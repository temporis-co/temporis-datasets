# temporis-datasets

A standard for preparing time series datasets for Temporis.

## Format

A dataset is a Hugging Face `DatasetDict` with two splits, `train` and
`test`. Every row is one **example**: a fixed length of one or more time
series, serialized as text with
[tstokenizers](https://github.com/temporis-ai/tstokenizers).

| column           | type     | meaning                                                                               |
|------------------|----------|---------------------------------------------------------------------------------------|
| `serialized`     | `string` | The example as text, patch by patch.                                                  |
| `patch_text_len` | `int`    | Characters per patch, so `len(serialized) / patch_text_len` is the number of patches. |

Rules:

1. `len(serialized)` is an exact multiple of `patch_text_len`.
2. Train and test do not overlap: a test example never shares a time step of a
   series with a train example.
3. No other columns.

### Events

A dataset whose series are affected by known events adds two columns:

| column      | type           | meaning                                                                                        |
|-------------|----------------|------------------------------------------------------------------------------------------------|
| `events`    | `list[string]` | Event texts in time order. Empty when nothing happened in the example.                         |
| `positions` | `list[int]`    | Where each event sits: the number of patches before it. `positions[i]` belongs to `events[i]`. |

An example row, 6 patches of 80 characters:

```python
{'serialized': '967>+=34.)0 /80-/40 /=0%/;0 /?0"/=0 /?0!...',   # 480 characters
 'patch_text_len': 80,
 'events': ['sine speed up'],
 'positions': [5]}
```

The event sits between patch 4 and patch 5, so it is seen from the last patch
on. Rules:

1. `len(events) == len(positions)`.
2. `0 <= position < number of patches`. Positions are strictly increasing:
   at most one event per position; changes that coincide are combined into
   one text.
3. No event means `events == []` and `positions == []`, never a placeholder
   such as `"none"`.

## Preparing a dataset

```
time series ─▶ fill gaps ─▶ cut patches ─▶ cut examples ─▶ serialize ─▶ split
```

1. **Time series.** Take each series as the source measures it, on its own
   fixed interval, the **grid**: one value per **time step**.
2. **Fill gaps.** Fill short gaps with the last value before them; leave long
   gaps empty, and build no example across them.
3. **Cut patches.** Cut each series into **patches** of `patch_len` time
   steps. Series that belong together, such as the vitals of one patient,
   are cut together and form one multivariate time series.
4. **Cut examples.** Cut **examples** of `num_patches` consecutive patches,
   starting every `stride` patches.
5. **Serialize.** Encode each example as text with
   `tstokenizers.Serializer(quantiles, patch_len, quant_scale)`, which
   normalizes and clips each series by the quantiles given.
6. **Split.** Choose a **cut** in time. Examples that end on or before it are
   train, examples that start on or after it are test.

A dataset with events adds one step: for each example, list the events that
fall inside it, each as a short text and the position of the patch boundary
where it takes effect.
