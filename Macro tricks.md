
### STRINGIFY

Definition:
```objc
#define xstr(s) str(s)
#define str(s) #s
```
Example:
```objc
#define foo 4
str (foo)
     → "foo"
xstr (foo)
     → xstr (4)
     → str (4)
     → "4"
```

### Nested if followed by else 
```objc
// The GNU compiler emits a warning if nested "if" statements are followed by
// an "else" statement and braces are not used to explicitly disambiguate the
// "else" binding.  This leads to problems with code like:
//
//   if (gate)
//     ASSERT_*(condition) << "Some message";
//
// The "switch (0) case 0:" idiom is used to suppress this.

#ifdef __INTEL_COMPILER
# define GTEST_AMBIGUOUS_ELSE_BLOCKER_
#else
# define GTEST_AMBIGUOUS_ELSE_BLOCKER_ switch (0) case 0: default:  // NOLINT
#endif
```
