# Reverse Shuffle Merge

贪婪算法的高阶应用, 专家级难度字符串操作. 算法详情请查看`HackerRank`:[原文 link](https://www.hackerrank.com/challenges/reverse-shuffle-merge/problem)

## Description

Given a string, `A`, we define some operations on the string as follows:

- `reverse(A)` denotes the string obtained by reversing string `A`. Example: `reverse("abc") = "cba"`
- `shuffle(A)` denotes any string that's a permutation of string `A`. Example: `shuffle("god") = {"god", "gdo", "ogd", "odg", "dog", "dgo"}`
- `merge(A1,A2)` denotes any string that's obtained by interspersing the two strings `A1` & `A2`, maintaining the order of characters in both. For example, `A1 = "abc"` & `A2 = "def"`, one possible result of `merge(A1,A2)` could be `abcdef`, another could be `abdecf`, another could be `adbecf` and so on.

Given a string `s` such that `s = merge(reverse(A), shuffle(A))` for some string `A`, find the lexicographically smallest `A`.

For example, `s=abab`. We can split it into two strings of `ab`. The reverse is `ba` and we need to find a string to shuffle in to get `abab`. The middle two characters match our reverse string, leaving the `a` and `b` at the ends. Our shuffle string needs to be `ab`. Lexicographically `ab < ba`, so our answer is `ab`.

### Function Description

Complete the reverseShuffleMerge function in the editor below. It must return the lexicographically smallest string fitting the criteria.

reverseShuffleMerge has the following parameter(s):

- `s` : a string

### Input Format

A single line containing the string `s`.

### Constraints

- `s` contains only lower-case English letters, ascii[a-z]
- `1 <= |s| <= 10000`

### Output Format

Find and return the string which is the lexicographically smallest valid `A`.

## Sample

### Sample Input

```java
abcdefgabcdefg
```

### Sample Output

```java
agfedcb
```

### Explanation

Split the string into two strings with like characters: `abcdefg` and `abcdefg`.

Reverse `agfedcb` = `bcdefga`

Shuffle `agfedcb` can be `abcdefg`.

Merge to `a**bcdefga**bcdefg`.

## Thinking space

For guys looking for simpler solution with low complexity, here's my submission (40 lines of code in C and 0s execution time for all test cases).

If the input string is S and the required answer is A, then the basic idea is as follows :

1) Store the count of each character (a to z) in S.

2) Update the count of characters required for A by dividing by 2.

3) Select each character for A by parsing S from right to left.

4) One "can" afford to "not select" a character, if its required-count for A will be fulfilled even after leaving it.

5) Considering point 4, always select smallest character if possible.

6) If a character can't be left (point 4 invalid), select the smallest character seen so far (even if it is not smallest for the entire string).

7) Ignore characters not required for A anymore

--- 摘自[AbhishekVermaIIT](https://www.hackerrank.com/AbhishekVermaIIT)

## Solution

```java
//LeftChars store all chars in the string, neededChars store result chars.
private static int[] leftChars = new int[26], neededChars = new int[26];
//Every time when we search best char, we store skipped chars in here.
private static int[] skippedChars;


static String reverseShuffleMerge(String s) {
    // forget about reverse - now merge is done on original string and its shuffle
    s = new StringBuilder(s).reverse().toString();

    //Store result in StringBuilder.
    StringBuilder sb = new StringBuilder();
    //Read all char to leftChars.
    createLeftChars(s);
    //We just need half of all chars.
    createNeededChars();

    //Auxiliary variable, lastIndex store each loop start index
    int lastIndex = -1;
    int currentIndex;
    char currentChar;
    int bestIndex;
    char bestChar;

    //Each loop, we search the smallest char for result's elements.
    while (sb.length() < s.length() / 2) {
        //26 english alphabet.
        skippedChars = new int[26];

        currentIndex = lastIndex + 1;
        bestIndex = lastIndex;
        bestChar = '{';

        do {
            currentChar = s.charAt(currentIndex);
            //If we need it and it's less than bestChar
            if (neededChars[currentChar - 'a'] > 0 && currentChar < bestChar) {
                bestChar = currentChar;
                bestIndex = currentIndex;
            }
            skipChar(currentChar);
            currentIndex++;
        } while(canBeSkipped(currentChar));

        sb.append(bestChar);
        //Remove elements before best char, because the sb is sequential
        //These elements before best char, are unreachable.
        for (int i = lastIndex + 1; i <= bestIndex; i++) {
            leftChars[s.charAt(i) - 'a']--;
        }
        neededChars[bestChar - 'a']--;
        lastIndex = bestIndex;
    }
    return sb.toString();
}

private static void createLeftChars(String s) {
    for (int i = 0; i < s.length(); i++) {
        char currentChar = s.charAt(i);
        leftChars[currentChar - 'a']++;
    }
}

private static void createNeededChars() {
    for (int i = 0; i < 26; i++) {
        neededChars[i] = leftChars[i] / 2;
    }
}

private static void skipChar(char currentChar) {
    skippedChars[currentChar - 'a']++;
}

private static boolean canBeSkipped(char currentChar) {
    int position = currentChar - 'a';
    return (leftChars[position] - skippedChars[position] - neededChars[position]) >= 0;
}
```
