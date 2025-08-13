# Regular Expressions
## Match the Start and End of a Line

The regular expression would match the search string anywhere on the line on which it occurred: the beginning, middle, or end of the word or line. Use a **line anchor** metacharacter to control where on a line to look for a match.

To match only at the beginning of a line, use the caret character (`^`). To match only at the end of a line, use the dollar sign (`$`).

With the same file as for the previous example, the `^cat` regular expression would match two lines.
```
cat
category
```

The `cat$` regular expression would find only one match, where the `cat` characters are at the end of a line.
```
cat
```

## Basic and Extended Regular Expression
The two types of regular expressions are **basic** regular expressions and **extended** regular expressions.

One difference between basic and extended regular expressions is in the behavior of the `|`, `+`, `?`, `(`, `)`, `{`, and `}` special characters.
- In basic regular expression syntax, these characters have a special meaning **only if they are prefixed** with a backslash `\` character.
- In extended regular expression syntax, these characters are special **unless they are prefixed** with a backslash `\` character. 
- Other minor differences apply to how the `^`, `$`, and `*` characters are handled.

The `grep`, `sed`, and `vim` commands use basic regular expressions. The `grep` command `-E` option, the `sed` command `-E` option, and the `less` command use extended regular expressions.

**Basic and Extended Regular Expression Syntax**

| Basic syntax  | Extended syntax | Description                                                                                                                                         |
| :------------ | :-------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------- |
| .             |                 | The period (`.`) matches any single character.                                                                                                      |
| ?             |                 | The preceding item is optional and is matched at most once.                                                                                         |
| *             |                 | The preceding item is matched zero or more times.                                                                                                   |
| +             |                 | The preceding item is matched one or more times.                                                                                                    |
| \\{`n`\\}     | {`n`}           | The preceding item is matched exactly `n` times.                                                                                                    |
| \\{`n`,\\}    | {`n`,}          | The preceding item is matched `n` or more times.                                                                                                    |
| \\{,`m`\\}    | {,`m`}          | The preceding item is matched at most `m` times.                                                                                                    |
| \\{`n`,`m`\\} | {`n`,`m`}       | The preceding item is matched at least `n` times, but not more than `m` times.                                                                      |
| [:alnum:]     |                 | Alphanumeric characters: `[:alpha:]` and `[:digit:]`; in the 'C' locale and ASCII character encoding, this expression is the same as `[0-9A-Za-z]`. |
| [:alpha:]     |                 | Alphabetic characters: `[:lower:]` and `[:upper:]`; in the 'C' locale and ASCII character encoding, this expression is the same as `[A-Za-z]`.      |
| [:blank:]     |                 | Blank characters: space and tab.                                                                                                                    |
| [:cntrl:]     |                 | Control characters. In ASCII, these characters have octal codes 000 through 037, and 177 (DEL).                                                     |
| [:digit:]     |                 | Digits: `0 1 2 3 4 5 6 7 8 9`.                                                                                                                      |
| [:graph:]     |                 | Graphical characters: `[:alnum:]` and `[:punct:]`.                                                                                                  |
| [:lower:]     |                 | Lowercase letters; in the 'C' locale and ASCII character encoding: `a b c d e f g h i j k l m n o p q r s t u v w x y z`.                           |
| [:print:]     |                 | Printable characters: `[:alnum:]`, `[:punct:]`, and space.                                                                                          |
| [:punct:]     |                 | Punctuation characters; in the 'C' locale and ASCII character encoding: `! " # $ % & ' ( ) * + , - . / : ; < = > ? @ [ \ ] ^ _ ' { \| } ~`.         |
| [:space:]     |                 | Space characters: in the 'C' locale, it is tab, newline, vertical tab, form feed, carriage return, and space.                                       |
| [:upper:]     |                 | Uppercase letters: in the 'C' locale and ASCII character encoding, it is: `A B C D E F G H I J K L M N O P Q R S T U V W X Y Z`.                    |
| [:xdigit:]    |                 | Hexadecimal digits: `0 1 2 3 4 5 6 7 8 9 A B C D E F a b c d e f`.                                                                                  |
| \b            |                 | Match the empty string at the edge of a word.                                                                                                       |
| \B            |                 | Match the empty string provided that it is not at the edge of a word.                                                                               |
| \\<           |                 | Match the empty string at the beginning of a word.                                                                                                  |
| \\>           |                 | Match the empty string at the end of a word.                                                                                                        |
| \w            |                 | Match word constituent. Synonym for `[_[:alnum:]]`.                                                                                                 |
| \W            |                 | Match non-word constituent. Synonym for `[^_[:alnum:]]`.                                                                                            |
| \s            |                 | Match white space. Synonym for `[[:space:]]`.                                                                                                       |
| \S            |                 | Match non-white space. Synonym for `[^[:space:]]`.                                                                                                  |
