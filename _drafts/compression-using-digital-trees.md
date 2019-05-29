---
title: TIL Compression using Digital Trees
subtitle: Let's build a brand new compression algorithm. Because why not...?
gh-repo: ThatRobVK/TIL-Compression-using-Digital-Trees
gh-badge: [star, fork, follow]
tags: [dotnet core]
---

Today I Learned how to build a new text compression algorithm in .NET Core using Digital Trees. I have done a tonne of .NET Framework back in the day, but Core is new to me. Let's see if I can notice any different.

   | has anyone done this before? Good to acknowledge if another algorithm already does this

## The Goal

I was reading [Rob Conery's The Imposter's Handbook](https://bigmachine.io/products/the-imposters-handbook/) and learned about Digital Trees (aka tries). They're a way to index text and allow very quick lookup of any text and traverse whatever text has been seen before to follow. It's the thing that drives predictive search and the like. It looks something like this:

   | Find image of digital tree

Compression algorithms have always fascinated me, although I have never taken the time to actually look into how they work. But when I saw this, I immediately saw a very compact way to store text with very little repetition. So that's what I set out to do: create an algorithm that can compress any length of text using digital trees and decompress it again to its original form.

## The Journey

### The basic compression design

I had a dull train journey from Coventry to London where I couldn't get my laptop out, so I had some time to think. The basic jist of my idea is to parse a text, building up a digital tree that contains all the words. Each letter has an ID and a link back to the previous letter in the sequence. The compressed text then only has to record the ID of the last letter of the word.

So that means the file contains:

```text
digital tree (probably letter followed by previous ID in sequence)
separator
sequence of ID's
```

There are further optimisations we can make, for sure. For example if we work out how many bits are used in the largest ID, then we can remove any redundant leading zero's before the sequence of ID's is stored. Even with a few thousand words you may well save kilobytes in the final file. But let's get the basic thing working before we worry about that.

### Let's get started

I've installed .NET Core SDK 2.2 and created a new console app.

```shell_session
$ dotnet new console
```

The first thing we need to sort out is the data structure we'll use in the app. For the trie we need a list of letters. In order to go back up the chain and recompose the words, they also need the ID of the previous letter. While we're building up the list, we also need to know which letters are already in the list forward, so we don't replicate the same sequence. Here's a sketch:

   | create sketch of id chain, forward looking and backward looking

So let's start with something simple, a struct that holds a character, an int and a list of ints, and stuff that struct into a generic list. It's not the neatest, but let's check it works first. I was going to rely on array index for ID, but if we need a list then we'll need to add an ID to the struct.

Without testing it yet, I've ended up with this:

```csharp
internal struct TrieNode
{
    internal int Id {get; private set;}
    internal char Character {get; private set;}
    internal int PreviousId {get; private set;}
    internal List<int> ChildIds {get; private set;}

    internal TrieNode(int id, char character, int previousId)
    {
        this.Id = id;
        this.Character = character;
        this.PreviousId = previousId;
        this.ChildIds = new List<int>();
    }

    internal void AddChild(int childId)
    {
        ChildIds.Add(childId);
    }
}
```

My thinking at this point is that we need to read in some text, split it into words and start populating the trie. Digging deeper, we need to compress all characters, but we only want to build sequences of words. My thinking is we just loop over every character in the text. We could track if we're currently handling a word and keep adding to it, or handle whitespace or special characters, and start new words when a new letter appears. Let's only worry about letters and spaces for now.

The basic encryptor is as follows. You take some text, turn it into a `char[]`, iterate over the characters and handle each.

```csharp
internal class Encrypter
{
    private List<TrieNode> trie; // the digital tree
    private List<int> trieIndices; // indices in the trie where words end

    internal Encrypter()
    {
        this.trie = new List<TrieNode>();
        this.trie.Add(new TrieNode(' ', 0));
        this.trieIndices = new List<int>();
    }

    internal string Encrypt(string input)
    {
        var lastTrieIndex = 0; // index of the previous trie node
        char[] inputChars = input.ToCharArray();

        // Loop through all characters in the input
        for (var i = 0; i < inputChars.Length; i++)
        {
            // Handle character
        }

        // Test output
        return "Trie: " + string.Join(" - ", trie) + "\nIndices: " + string.Join(", ", trieIndices);
    }
}
```

So let's think about the processing. Right off each character is either a letter or a space. The root of the trie is a space, so for any space we just add 0 to the `trieIndices` list. For a letter there are two options: either the parent already contains this letter as a child, or it doesn't. So we need to look at the parent, iterate over its children, see if we find the current character. If we do then move on to the next character, but if we don't then we need to add the current character as a child of the parent. Lastly, if this is the last letter of the current word, then add the index of the current character to `trieIndices`.

After a few rounds of debugging silly typos and mistakes, this is what I ended up with. This block goes inside the `for` loop above.

```csharp
var currentChar = inputChars[i];

if (Char.IsLetter(currentChar))
{
    bool found = false;

    // Search for the current char in the trie
    foreach (int childIndex in trie[lastTrieIndex].ChildIndices)
    {
        if (trie[childIndex].Character.Equals(currentChar))
        {
            // Found it, so set the index to the child and move on to the next character
            lastTrieIndex = childIndex;
            found = true;
        }

        if (found) break;
    }

    if (!found)
    {
        // Not found - add a new trie node and add bi-directional links with parent
        var trieNode = new TrieNode(currentChar, lastTrieIndex);
        trie.Add(trieNode);
        trie[lastTrieIndex].AddChild(trie.Count - 1);

        // Set the last index to the new item's index
        lastTrieIndex = trie.Count - 1;
    }

    // Look ahead, if this is the last character or the next character isn't a letter, then add the word to the list
    if (inputChars.Length == i + 1 || !Char.IsLetter(inputChars[i+1]))
    {
        trieIndices.Add(lastTrieIndex);
    }
}
else
{
    // Add a space to the list and reset to index 0
    trieIndices.Add(0);
    lastTrieIndex = 0;
}
```

{: .box-note}
This code could be cleaned up quite a lot, for example using Linq for searching instead of iteration. For now I'm focusing on making this work and easy to follow. At the end we'll go back and refactor this.

