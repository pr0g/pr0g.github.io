---
layout: post
title:  "Column Major and Row Major Vectors and Matrices (for games)"
date:  2022-12-24 14:52:13 +0000
categories: mathematics matrix
---
This is yet another post about understanding the difference between column major and row major vectors and matrices (in a game development setting). If you're attempting to learn or remember the difference between them then hopefully this will be a useful resource and can supplement other reading.

This article will touch on some of the theory but the main focus of what is to follow will be on what crops up when implementing a maths library for games. For a really fantastic in-depth treatment on the subject of matrices check out these chapters from [Game Math](https://gamemath.com/).

- [Chapter 4: Introduction to Matrices](https://gamemath.com/book/matrixintro.html)
- [Chapter 6: More on Matrices](https://gamemath.com/book/matrixmore.html)

## Prerequisite

When we talk about an `M x N` matrix, in the vast majority of cases (throughout textbooks, papers etc..) the order is rows (`M`) and columns (`N`). This is by far the most common approach and I'd be tempted to recommend sticking with this. There are reasons why deciding to refer to columns first has some advantages (tied to storage reason's we'll get to later), but for consistency, row/column should (in my opinion) generally be preferred.

## Convention and Storage/Access

There are two pieces to this puzzle, convention (the maths part (theory)) and storage/access (the computer science part (practice)). They are both intertwined but it's possible to deal with them individually. Let's start with convention.

> If you're already comfortable with the convention part the feel free to skip it and go to [storage](#storageaccess).

### Convention

When we write a vector we can either stand it upright (column vector) or lie it flat (row vector).

```c++
// column vector with 4 elements
 c
[1] r
[2] r
[3] r
[4] r

// row vector with 4 elements
 c  c  c  c
[1][2][3][4] r
```

For purposes of the convention part, when we talk generally about multiplying matrices together, we can think of the above as either a 4x1 matrix (4 rows, 1 column - column major) or a 1x4 matrix (1 row, 4 columns - row major). This fits with the bigger picture of multiplying matrices of different sizes together (this is useful to know about but rarely something that comes up outside of multiply vectors and matrices, at least in a game development setting).

To multiply two matrices together we multiply the row(s) on the left hand side with the column(s) on the right hand side.

For a matrix multiplication to be valid, the number of columns of the matrix on the left has to equal the number of rows in the matrix on the right. Most of the time in game engines we're just multiplying square matrices (`3 x 3` or `4 x 4`) so we don't think about it, but if we remember to sometimes think of vectors as their `1 x N` or `M x 1` matrix counterparts, this explains why when multiplying by a column vector it goes on the right and when multiplying by a row vector it goes on the left.

```c++
// a 4x4 matrix we'd like to use to multiply a column vector
 c  c  c  c      c
[a][e][i][m] r  [1] r
[b][f][j][n] r  [2] r
[c][g][k][o] r  [3] r
[d][h][l][p] r  [4] r

// left hand side (lhs) - columns == 4, rows == 4
// right hand side (rhs) - columns == 1, rows == 4,
// lhs columns (4) and rhs rows (4) match!
```

```c++
// a 4x4 matrix we'd like to use to multiply a row vector
 c  c  c  c      c  c  c  c
[1][2][3][4] r  [a][b][c][d] r
                [e][f][g][h] r
                [i][j][k][l] r
                [m][n][o][p] r

// lhs - columns == 4, rows == 1
// rhs - columns == 4, rows == 4
// lhs columns (4) and rhs rows (4) match!
```

This rule satisfies the requirement that we always multiply the row on the left hand side with the column on the right hand side. When multiplying matrices together the size of the resulting matrix always has the same number of rows from the left matrix and the same number of columns from the right matrix. This is why when we multiply a row vector by matrix we get a row vector back, and when we multiply a column vector by a matrix we get a column vector back.

```c++
// 4 rows, 4 columns   4 rows, 1 column
 c  c  c  c             c
[a][e][i][m] r         [1] r
[b][f][j][n] r         [2] r
[c][g][k][o] r         [3] r
[d][h][l][p] r         [4] r

// result
[a1 + e2 + i3 + m4]
[b1 + f2 + j3 + n4]
[c1 + g2 + k3 + o4]
[d1 + h2 + l3 + p4]

// 1 row, 4 columns    4 rows, 4 columns
 c  c  c  c             c  c  c  c
[1][2][3][4] r      x  [a][b][c][d] r
                       [e][f][g][h] r
                       [i][j][k][l] r
                       [m][n][o][p] r

// result
[1a + 2e + 3i + 4m][1b + 2f + 3j + 4n][1c + 2g + 3k + 4p][1d + 2h + 3l + 4p]
```

The last relevant part of the theory to cover is what we mean when we say post-multiply or pre-multiply. This is relevant because the order we multiply matrices is important. Because of how we traverse rows on the left side and columns on the right side, if we swap the matrices then we'll get a different result out.

When we multiply two column major matrices together we post-multiply them which just means we start on the right and work our way left. For transformations we usually want to scale first, then rotate and finally translate, this would look like:

```c++
// concatenating transforms (column major matrices)
transform = translate * rotate * scale;
// transforming a column vector
transformed_point = transform * point;
```

When dealing with row major matrices we pre-multiply them which means we start on the left and work our way to the right. For the scale, rotate and translate transformation before it now looks like this:

```c++
// concatenating transforms (row major matrices)
transform = scale * rotate * translate;
// transforming a row vector
transformed_point = point * transform;
```

When it comes to matrices that's (in my mind at least) the stuff that comes up most often. If you want to learn more do checkout the links mentioned at the top of the article. This is all just theory though, what actually happens when you want to implement these concepts in your very own math library.

### Storage/Access

When deciding how to store your matrix data there are quite a few options, all with various pros and cons.

#### Array

```c++
struct mat3_t {
  float data[9];
};
```

This approach is maybe the most flexible but comes with some considerations. The first and most important is how to interpret the data.

##### Array Option #1

Each 3 elements when traversed are the rows of the matrix

```c++
[0][1][2][3][4][5][6][7][8] // data

// conceptually becomes
[0][1][2]
[3][4][5]
[6][7][8]
```

##### Array Option #2

Each 3 elements when traversed are the columns of the matrix

```c++
[0][1][2][3][4][5][6][7][8] // data

// conceptually becomes
[0][3][6]
[1][4][7]
[2][5][8]
```

The key thing with the two approaches above is that the storage is identical. The part that's different is access.

To make accessing elements by their row/column we can write an accessor for either the column major case or row major case.

```c++
// row major (3x3 matrix)
int row_col_rm(int row, int col) {
  return row * 3 + col;
}

mat3_t mat3 = { ... };
int offset = row_col_rm(1, 2); // offset 5
int value = mat[offset];       // value 6

// col major (3x3 matrix)
int row_col_cm(int row, int col) {
  return col * 3 + row;
}

mat3_t mat3 = { ... };
int offset = row_col_cm(1, 2); // offset 7
int value = mat[offset];       // value 8
```

The bit where this gets hard/confusing is if you want to start writing out matrices in your library (this is common for things like rotation and projection matrices). Because the data is in a one dimensional array, and we're applying a convention that decides how we access elements, when we write out a matrix in C/C++, it will always look like it's in row major format even if we're using the column major convention.

```c++
               [ col 0 ] [ col 1 ] [ col 2 ]
mat3_t mat3 = { 0, 1, 2,  3, 4, 5,  6, 7, 8 };

// reformatted
mat3_t mat3 = { 0, 1, 2,   // [ col 0 ]
                3, 4, 5,   // [ col 1 ]
                6, 7, 8 }; // [ col 2 ]

// the above can be confusing...
```

Access is the thing that makes the difference. If using row major, we walk each row on the left multiplying by each column on the right. If however we've decided to use the column major convention with this format, we need to update the traversal order.

One (not particularly ideal) solution is to keep the traversal the same, transpose both matrices, swap the order, multiply the and then transpose the result once more. This works, but it's a bunch of extra work. One fairly neat solution is to use our `row_col` lookup functions.

```c++
// row major
mat3_t operator*(const mat3_t& lhs, const mat3_t& rhs) {
    mat3_t m;
    for (int r = 0; r < 3; ++r) {
        for (int c = 0; c < 3; ++c) {
            float elem = 0.0f;
            for (int s = 0; s < 3; ++s) {
                elem += lhs.data[row_col_rm(r, s)]
                        rhs.data[row_col_rm(s, c)];
            }
            m.data[row_col_rm(r, c)] = elem;
        }
    }
    return m;
}

// column major
mat33 operator*(const mat33& lhs, const mat33& rhs) {
    mat33 m;
    for (int r = 0; r < 3; ++r) {
        for (int c = 0; c < 3; ++c) {
            float elem = 0.0f;
            for (int s = 0; s < 3; ++s) {
                elem += lhs.data[row_col_cm(r, s)] * // changed
                        rhs.data[row_col_cm(s, c)];  // changed
            }
            m.data[row_col_cm(r, c)] = elem;         // changed
        }
    }
    return m;
}
```

The only difference is swapping out `row_col_rm` with `row_col_rm` and making sure we swap the order of the two matrices. If we print the results of the `data` member we'll see we get the exact same result.

```c++
// debug printing functions
for (int e = 0; e < 9; ++e) {
  std::cout << result.data[e] << ' ';
}

for (int r = 0; r < 3; ++r) {
  for (int c = 0; c < 3; ++c) {
    std::cout << result.data[row_col_rm(r, c)] << ' '; // row
    // or
    std::cout << result.data[row_col_cm(r, c)] << ' '; // column
  }
  std::cout << '\n';
}

// row major
mat3_t a = {1, 2, 3, 4, 5, 6, 7, 8, 9};
mat3_t b = {9, 8, 7, 6, 5, 4, 3, 2, 1};
mat3_t result = a * b; // using row_col_rm version

// prints
// 30 24 18 84 69 54 138 114 90
//
// 30 24 18
// 84 69 54
// 138 114 90
//
// display order in rows

// column major
mat3_t a = {1, 2, 3, 4, 5, 6, 7, 8, 9};
mat3_t b = {9, 8, 7, 6, 5, 4, 3, 2, 1};
mat3_t result = b * a; // using row_col_cm version

// prints
// 30 24 18 84 69 54 138 114 90
//
// 30 84 138
// 24 69 114
// 18 54 90
//
// display order in columns
```

One other advantage to this approach is basis vectors (axes and position) are contiguous in memory (we see `[x, y, z, x, y, z, x, y...]` etc...). This is often what graphics APIs such as OpenGL and DirectX expect when passing data to shaders.

##### Array Option #3

Interleave basis vectors and use column major

```c++
[0][3][6][1][4][7][2][5][8] // data

// conceptually becomes
[0][3][6]
[1][4][7]
[2][5][8]
```

In this approach we explicitly lay the data out in column major format but use the lookup and traversal we'd used when the storage was row major.

One big advantage to this approach is it's possible to write column major matrices out in the source code.

```c++
mat3_t a = {1, 4, 7,
            2, 5, 8,
            3, 6, 9};

// or for a translation matrix
mat4_t transform = { 1, 0, 0, tx,
                     0, 1, 0, ty,
                     0, 0, 1, tz,
                     0, 0, 0, 1 };
```

We also continue to use `row_col_rm` (or simply just `row_col`) as we're explicitly using the column storage order in how we layout the data. One downside to be aware of with this approach is the basis vectors are no longer contiguous in memory which might make certain access patterns slower (though this likely negligible in practice) and you'll need to transpose the matrix before sending it to a graphics api (e.g. as a uniform) to ensure the data is contiguous and no longer interleaved.

```c++
mat3_t a = {1, 4, 7,
            2, 5, 8,
            3, 6, 9};
mat3_t b = {9, 6, 3,
            8, 5, 2,
            7, 4, 1};
mat3_t result = b * a;

// prints
// 30 84 138 24 69 114 18 54 90
//
// 30 84 138
// 24 69 114
// 18 54 90
//
// display order columns
```

If we transpose `result` we get back the matrix in row major format

```c++
mat3_t transposed = transpose(result);

// prints
// 30 24 18 84 69 54 138 114 90
//
// 30 24 18
// 84 69 54
// 138 114 90
```

#### Multi-dimensional array

```c++
struct mat3_t {
  float data[3][3];
};
```



{% highlight cpp %}
void doSomething()
{

}
{% endhighlight %}
