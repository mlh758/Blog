+++
title = "Unintended Consequences of Cumbersome APIs"
date = "2024-08-31"
in_search_index = true
+++


I recently found myself working an issue on the front end of one of our applications. The functionality I was looking into involved
a form building screen where you define questions for a user to complete. You can drag and drop questions to change the order. If you
click on one of the questions it expands so you can configure details about the question such as the type of responses accepted. The problem
I was investigating was that sometimes if you dragged and released an item it would get stuck outside of the list and become
unresponsive from that point onward.

When I started testing the application I made a form and started dragging items around. I noticed that the page slowed down quite a bit
as I moved items and if I jiggled an item quickly it tended to get stuck outside of the form. I also noticed that if I opened
one of the questions and tried to modify it, every key stroke was delayed. Using the React profiling tools it was taking about 100ms
to render each key stroke on my laptop.

The first thing I wanted to fix was the lag/jitter trying to drag items. Every time one of the dragged elements crossed another
triggering an order change there was a noticeable delay. Everything involved in the form builder was re-rendering. At
this point I usually take a look at the library APIs and see if I can minimize what is actually being updated to limit the fallout.
The form builder was created using `react-hook-form`. Since the form is a dynamic list of questions it was using the `useFieldArray`
hook specifically. We were updating the fields with `replace` which as you might assume replaces the entire contents of the array.
There is also a `move` function returned by `useFieldArray` which seemed like a natural fit given that we were moving one item
elsewhere in the array. This should cause fewer updates in the tree and fewer renders.

## Never that easy

My good luck ran out there. To do the animations we were using the Framer library's `Reorder.Group`. To notify your application
of a change in order it provides a callback `onReorder` which just hands you back the new list of items. This is why we were using
`replace` instead of the more limited `move`. It also lead to a lot of complexity in the surrounding component since we needed to react
to specific changes in ordering which lead to a fair amount of complex code trying to keep the old state around and compare it to any
new state coming out. This also meant some extra memoization, copying props into state, and `useEffects` to try and reconcile it all together.
The simple API just wasn't giving us the information we needed and it put quite a bit of additional burden on the consuming component
instead.

I was convinced there had to be an easier way to get this information but Framer doesn't provide any other callbacks that I could find and I really wanted to get this complexity out of the component while being able to use the narrower `move` function from `useFieldArray`.

## I wonder if Copilot will make this easy

My first thought was to do something simple like iterate over the new array comparing the order to the old one. When I find the changed element use `findIndex` on the old array and done. I knew right away this was too simplistic since whichever direction I iterated from reordering up or down the array would make one of those look like too many things had changed. This seemed like a common enough problem that I could probably get an LLM to make this function for me. This was only the first step in trying to optimize a pretty deep component tree and I didn't want to spend a lot of time on it.

### Hey that's just my bad idea again

I used this prompt:

```
Given two arrays of IDs where a single element was moved to a new location in the array,
determine which element was moved and to which location.

Also tried: Given two arrays A and B where A is the original state and B is the array
after moving one item to another location in the array,
provide a function for determining which element was moved.
```

I received this code in response:

```
function findMovedElement(originalArray, newArray) {
  let movedElement = null;
  let originalIndex = -1;
  let newIndex = -1;

  // Create a map to store the indices of elements in the original array
  const originalIndexMap = new Map();
  originalArray.forEach((element, index) => {
    originalIndexMap.set(element, index);
  });

  // Iterate through the new array to find the moved element
  for (let i = 0; i < newArray.length; i++) {
    const element = newArray[i];
    if (originalIndexMap.has(element)) {
      const originalIdx = originalIndexMap.get(element);
      if (originalIdx !== i) {
        movedElement = element;
        originalIndex = originalIdx;
        newIndex = i;
        break;
      }
    }
  }

  return { movedElement, originalIndex, newIndex };
}
```

This is basically what I had envisioned doing initially and knew couldn't be correct. Here's an initial test:

```
let a = [1, 2, 3, 4, 5]
let b = [1, 3, 2, 4, 5]
findMovedElement(a, b) // {movedElement: 3, originalIndex: 2, newIndex: 1}
```

So far so good I suppose. But what about the other direction and not a trivial swap?

```
b = [2, 3, 4, 5, 1]
findMovedElement(a, b) // {movedElement: 2, originalIndex: 1, newIndex: 0}
```

Well that's not right. I dragged  `1` down to the end. `2` just seems like it moved because
the algorithm is only looking at the first mismatch and stopping. Then it hit me, drag and drop
is just insertion and deletion.

### This is a solved problem

I had recently read through the excellent _Algorithm Design Manual_ by Steven Skiena and it was
still on my desk. Why no, potential coworkers reading this, I'm not doing any interview prep.

*Move along.*

Section 10.2.2 is _Edit Distance by Dynamic Programming_. Given a sequence of values, what is
the minimum number of insertions, deletions, and substitutions needed to turn one sequence into the
other. With dynamic programming you end up with a table that records the steps taken to arrive
at the answer and so you can traverse that table to arrive at the information I was looking for.
The next section of the book does exactly this.

Unfortunately I'm also somewhat lazy and I didn't feel like piecing together the code from several examples
and translating it from C to JavaScript. I'm also really not supposed 
to be spending this much time on the issue. Now that I have a better idea of what I'm asking
for, let's try the LLM again.

### AI tries edit distance

This was my next prompt:

```
Given two arrays A and B where A is the original state and B
is the array after moving one item to another location in the
array, provide a function for determining which element was
moved using an edit distance algorithm such as Levenshtein
```

I was given this code in return:

```
function findMovedElementUsingEditDistance(A, B) {
  const m = A.length;
  const n = B.length;

  // Initialize the dp table
  const dp = Array.from({ length: m + 1 }, () => Array(n + 1).fill(0));

  // Fill in the base cases
  for (let i = 0; i <= m; i++) dp[i][0] = i;
  for (let j = 0; j <= n; j++) dp[0][j] = j;

  // Fill in the dp table
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (A[i - 1] === B[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1];
      } else {
        dp[i][j] = Math.min(dp[i - 1][j], dp[i][j - 1]) + 1;
      }
    }
  }

  // Trace back to find the moved element
  let i = m, j = n;
  let movedElement = null;
  let originalIndex = -1;
  let newIndex = -1;

  while (i > 0 && j > 0) {
    if (A[i - 1] === B[j - 1]) {
      i--;
      j--;
    } else if (dp[i][j] === dp[i - 1][j] + 1) {
      // Deletion in A
      i--;
    } else if (dp[i][j] === dp[i][j - 1] + 1) {
      // Insertion in B
      j--;
    } else {
      // Move operation
      movedElement = A[i - 1];
      originalIndex = i - 1;
      newIndex = j - 1;
      break;
    }
  }

  return { movedElement, originalIndex, newIndex };
}
```

Testing with our same `b` array as before:

```
let b = [2, 3, 4, 5, 1]
findMovedElementUsingEditDistance(a, b) // {movedElement: null, originalIndex: -1, newIndex: -1}
```

Well that's not right. Let's prompt a correction:

```
When I provide arrays [1, 2, 3, 4, 5] and [1, 3, 4, 5, 2] The answer should be { movedElement: 2, originalIndex: 1, newIndex: 4} but with that function the movedElement is null which is incorrect
```

When I did that it just stuck a variation of the first, already incorrect code at the end of the edit distance function and
called into that if the trace didn't return an answer. At least I didn't get `null` anymore I guess?

### I was making this too complicated

After moving on to work I as actually supposed to be doing for a while I realized I was asking
Copilot to generate code for a somewhat less common and more general algorithm. Longest Common
Subsequence is _also_ an edit distance function, and it's a simpler one that only supports
insert and deletion. There's probably more sample code for this floating around the internet.

Here is the prompt I used:

```
Given two arrays: const A = [1, 2, 3, 4, 5]; const B = [1, 3, 4, 5, 2];
Use longest common subsequence to show the single deletion and insertion required to turn A into B
I also need to know the indexes at which the insertions and deletions happen
```

I received this code in response:

```
function longestCommonSubsequence(A, B) {
  const m = A.length;
  const n = B.length;

  // Initialize the dp table
  const dp = Array.from({ length: m + 1 }, () => Array(n + 1).fill(0));

  // Fill in the dp table
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (A[i - 1] === B[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1] + 1;
      } else {
        dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
      }
    }
  }

  // Trace back to find the LCS
  let i = m, j = n;
  const lcs = [];

  while (i > 0 && j > 0) {
    if (A[i - 1] === B[j - 1]) {
      lcs.unshift({ value: A[i - 1], indexA: i - 1, indexB: j - 1 });
      i--;
      j--;
    } else if (dp[i - 1][j] >= dp[i][j - 1]) {
      i--;
    } else {
      j--;
    }
  }

  return lcs;
}

// Disclaimer - I modified this code to get a similar
// result format to the previous functions.
function findSingleEdit(A, B) {
  const lcs = longestCommonSubsequence(A, B);

  // Find the deletion
  const deletion = A.map((value, index) => ({ value, index }))
    .filter(x => !lcs.some(l => l.indexA === x.index));

  // Find the insertion
  const insertion = B.map((value, index) => ({ value, index }))
    .filter(x => !lcs.some(l => l.indexB === x.index));
  if (insertion.length === 0 || deletion.length === 0) return;

  const movedElement = deletion[0].value;

  return { movedElement, from: deletion[0].index, to: insertion[0].index };
}
```

Something I found interesting here is that my prompt is more broken down into two steps:

* Use LCS
* Tell me the edits

and in response Copilot gave me two separate functions. This one also seems to be correct.
I wrote some unit tests using a few variations of `b` and got the results I was looking for.

## What was I supposed to be doing again?

With a working LCS function in hand I could finally overcome the limited API of Framer and use the more appropriate edit function provided by `react-hook-form`. I was also able to clean up quite a bit of our own code that was trying to guess at this difference. Less code here also meant removing intermediate state and some `useEffect` from the component which made the whole thing easier to grok. More importantly to me remaining employed after this much tinkering, this helped with the performance of the drag and drop functionality. Since we had unique `key` props on the top level components being rendered in the list, the performance improvments likely came as a result of the simplified component code and eliminating so much `useState` and `useEffect` which would force renders.

The performance of editing the dynamic form components was addressed primarily with `useMemo`, `useCallback` and
refactoring components to limit the touch points of `useContext` so fewer things rendered on each keystroke. That
was quite a bit more work than the edit distance stuff, but it also probably isn't very interesting to read about.

In the end this made me miss Elixir which provides two [edit distance functions](https://hexdocs.pm/elixir/1.12/String.html#myers_difference/2) for strings in the standard library. It also provides them for the `List` type so I could have stayed in the standard library and never had to touch any of this. Unfortunately nobody seems interested in paying me to write Elixir or F# code.
