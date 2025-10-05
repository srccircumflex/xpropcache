# xpropcache

Lightweight utilities for cached properties with selective reset and pickle-safe classes.

- Extends [functools.cached_property](https://docs.python.org/3/library/functools.html#functools.cached_property) with flags
- Class decorator for automatic registration
- Selective cache invalidation
- Automatic cleaning of cache values and marked attributes for pickling

The **xpropcache** decorator is `@PropCache.cached_property | @PropCache.cached_property(flags)`

> _these properties are referred as **xprops** below_

## Installation

- Pure Python, no external dependencies except standard library
- Python 3.10+ required


```commandline
pip install xpropcache --upgrade
```

## Quickstart

```python
from xpropcache import PropCache, F__PickleIgnore__

FLAGS_DB = 0x01

@PropCache
class User:
    
    uid: int
    
    # ignore during pickling
    _session: Session | F__PickleIgnore__

    def __init__(self, uid, loader, session=None):
        self.uid = uid
        self._loader = loader
        self._session = session

    @PropCache.cached_property
    def profile(self):
        return self._loader.load_profile(self.uid)

    @PropCache.cached_property(FLAGS_DB)
    def settings(self):
        return self._loader.load_settings(self.uid)

u = User(42, loader)
if not u.profile:  # computed and cached
    new_user(u)
    PropCache.cp_purge(u)  # invalidates all xprops
elif u.settings.is_premium:  # computed and cached
    ...
    PropCache.cp_reset_by_flag(u, FLAGS_DB)  # only invalidates settings
```

## Concepts

- `@PropCache.cached_property | @PropCache.cached_property(flags)`: 
  - Like `functools.cached_property`, with additional optional flags (`int`) for grouping.
  - Cached values are ignored during pickling.
- `@PropCache`:
  - Decorator for classes. Registers **xprops**, extends `__getstate__`, and adds `__cp_purge__`.
- `F__PickleIgnore__`:
  - Marker type for type annotations. Attributes annotated with this type (directly or in union) 
    are removed during pickling (in addition to cached **xprops**).

## API (short)

- `PropCache.cp_purge(inst)`: 
  Reset all cached **xprops** of the instance.
- `PropCache.cp_reset_by_flag(inst, flags)`: 
  Reset cached **xprops** of the instance whose flag bits apply (`xprop.flags & flags`).
- `PropCache.pickle_purge(inst, inst.__dict__)`: 
  Removes cached **xprops** and attributes marked with `F_PickleIgnore__` from `__dict__`
- Classes automatically receive:
  - `__getstate__` with the pickle-purge logic
  - `__cp_purge__`: alias to `PropCache.cp_purge(self)`

## Notes

- Inheritance: **xprops** and `F_PickleIgnore__` flags are merged via the MRO.
