---
layout: post
title:  "Column Major and Row Major Vectors and Matrices (for games)"
date:  2022-12-26 14:52:13 +0000
categories: mathematics matrix
---
This is yet another post about understanding the difference between column major and row major vectors and matrices (in a game development setting). If you're attempting to learn or remember the difference between them then hopefully this will be a useful resource and can supplement other reading.

This article will touch on some of the theory but the main focus of what is to follow will be on what crops up when implementing a math library for games. For a really fantastic in-depth treatment on the subject of matrices check out these chapters from [Game Math](https://gamemath.com/).

- [Chapter 4: Introduction to Matrices](https://gamemath.com/book/matrixintro.html)
- [Chapter 6: More on Matrices](https://gamemath.com/book/matrixmore.html)

## Prerequisite

When we talk about an `MxN` matrix, in the vast majority of cases (throughout textbooks, papers etc..) the order is rows (`M`) and columns (`N`). This is by far the most common approach and I'd be tempted to recommend sticking with this. There are reasons why deciding to refer to columns first has some advantages (tied to storage reason's we'll get to later), but for consistency, row/column should (_in my opinion_) generally be preferred.

## Convention and Storage/Access

There are two pieces to this puzzle, convention (the maths part (_theory_)) and storage/access (the computer science part (_practice_)). They are both intertwined but it's possible to deal with them individually. Let's start with convention.

> If you're already comfortable with the convention part then feel free to skip it and go straight to [storage](#storageaccess).

### Convention

When we write a vector we can either stand it upright (column vector) or lie it flat (row vector).

{% raw %}

```c++
// c - column
// r - row

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

{% endraw %}

For purposes of the convention part, when we talk generally about multiplying matrices together, we can think of the above as either a `4x1` matrix (4 rows and 1 column - column major) or a `1x4` matrix (1 row and 4 columns - row major). This fits with the bigger picture of multiplying matrices of different sizes together (this is useful to know about but it's rarely something that comes up outside of multiplying vectors and matrices together, at least in a game development setting).

To multiply two matrices together we multiply the row(s) on the left hand side with the column(s) on the right hand side.

For a matrix multiplication to be valid, the number of columns the matrix has on the left has to equal the number of rows the matrix has on the right. Most of the time in game engines we're just multiplying square matrices (`3x3` or `4x4`) so we don't think about it, but if we remember to sometimes think of vectors as their `1xN` or `Mx1` matrix counterparts, this explains why when multiplying by a column vector it goes on the right and when multiplying by a row vector it goes on the left.

{% raw %}

```c++
// lhs - left hand side
// rhs - right hand side

// a 4x4 matrix we'd like to use to multiply a column vector
 c  c  c  c      c
[a][e][i][m] r  [1] r
[b][f][j][n] r  [2] r
[c][g][k][o] r  [3] r
[d][h][l][p] r  [4] r

// lhs: columns == 4, rows == 4
// rhs: columns == 1, rows == 4,
// lhs columns (4) and rhs rows (4) match!
```

{% endraw %}

{% raw %}

```c++
// a 4x4 matrix we'd like to use to multiply a row vector
 c  c  c  c      c  c  c  c
[1][2][3][4] r  [a][b][c][d] r
                [e][f][g][h] r
                [i][j][k][l] r
                [m][n][o][p] r

// lhs: columns == 4, rows == 1
// rhs: columns == 4, rows == 4
// lhs columns (4) and rhs rows (4) match!
```

{% endraw %}

This rule satisfies the requirement that we always multiply the row on the left hand side with the column on the right hand side. When multiplying matrices together the size of the resulting matrix always has the same number of rows from the left matrix and the same number of columns from the right matrix. This is why when we multiply a row vector by a matrix we get a row vector back, and when we multiply a column vector by a matrix we get a column vector back.

{% raw %}

```c++
// 4 rows, 4 columns   4 rows, 1 column
 c  c  c  c             c
[a][e][i][m] r         [1] r
[b][f][j][n] r         [2] r
[c][g][k][o] r         [3] r
[d][h][l][p] r         [4] r

// result (column vector)
[a1 + e2 + i3 + m4]
[b1 + f2 + j3 + n4]
[c1 + g2 + k3 + o4]
[d1 + h2 + l3 + p4]

// 1 row, 4 columns   4 rows, 4 columns
 c  c  c  c            c  c  c  c
[1][2][3][4] r        [a][b][c][d] r
                      [e][f][g][h] r
                      [i][j][k][l] r
                      [m][n][o][p] r

// result (row vector)
[1a + 2e + 3i + 4m][1b + 2f + 3j + 4n][1c + 2g + 3k + 4p][1d + 2h + 3l + 4p]

// note: the value of each corresponding element in the resulting vectors is the same
```

{% endraw %}

The last part of the theory (which is closely related to the above) is what we mean when we say post-multiply or pre-multiply. This is relevant because the order we multiply matrices is important. The way we traverse rows on the left side and columns on the right side means if we swap the matrices and perform the same operation (multiplying), then we'll get a different result. When we multiply two column major matrices together we post-multiply them which just means we start on the right and work our way left.

For a concrete example let's take transformations. We usually want to scale first, then rotate and finally translate, this would look like so:

{% raw %}

```c++
// concatenating transforms (column major matrices)
// order: scale, rotate, translate (happening right to left)
transform = translate * rotate * scale;
// transforming a column vector
transformed_point = transform * point;
```

{% endraw %}

When dealing with row major matrices we pre-multiply them which means we start on the left and work our way to the right. For the scale, rotate and translate transformation before it now looks like this:

{% raw %}

```c++
// concatenating transforms (row major matrices)
// order: scale, rotate, translate (happening left to right)
transform = scale * rotate * translate;
// transforming a row vector
transformed_point = point * transform;
```

{% endraw %}

When it comes to matrices that's (in my mind at least) the stuff that comes up most often. If you want to learn more please check out the links mentioned at the top of the article. This is all just theory though, let's review what actually happens when you want to implement these concepts in your very own math library.

### Storage/Access

When deciding how to store your matrix data there are quite a few options, all with various pros and cons.

#### Array

{% raw %}

```c++
struct mat3_t {
  float data[9];
};
```

{% endraw %}

This approach is maybe the most flexible but comes with some considerations. The first and most important is how to interpret the data.

#### Array Option #1

Each 3 elements when traversed are the rows of the matrix.

{% raw %}

```c++
[0][1][2][3][4][5][6][7][8] // data

// conceptually becomes
[0][1][2] // row 0
[3][4][5] // row 1
[6][7][8] // row 2
```

{% endraw %}

{% raw %}

```c++
               [ row 0 ] [ row 1 ] [ row 2 ]
mat3_t mat3 = { 0, 1, 2,  3, 4, 5,  6, 7, 8 };

// reformatted
mat3_t mat3 = { 0, 1, 2,   // row 0
                3, 4, 5,   // row 1
                6, 7, 8 }; // row 2
```

{% endraw %}

__[Godbolt Link](https://gcc.godbolt.org/z/sKanW5nfd)__

#### Array Option #2

Each 3 elements when traversed are the columns of the matrix.

{% raw %}

```c++
[0][1][2][3][4][5][6][7][8] // data

// conceptually becomes
//  c  c  c
//  o  o  o
//  l  l  l
//  0  1  2
   [0][3][6]
   [1][4][7]
   [2][5][8]
```

{% endraw %}

{% raw %}

```c++
               [ col 0 ] [ col 1 ] [ col 2 ]
mat3_t mat3 = { 0, 1, 2,  3, 4, 5,  6, 7, 8 };

// reformatted
mat3_t mat3 = { 0, 1, 2,   // col 0
                3, 4, 5,   // col 1
                6, 7, 8 }; // col 2

// the above can be confusing...
```

{% endraw %}

__[Godbolt Link](https://gcc.godbolt.org/z/EbYanPxq7)__

The key thing with the two approaches above is that the storage is identical. The part that's different is access/traversal.

To access elements by their row/column we can write a function for either the column major case or row major case.

{% raw %}

```c++
// row major (3x3 matrix)
int row_col_rm(int row, int col) {
  return row * 3 + col;
}

// col major (3x3 matrix)
int row_col_cm(int row, int col) {
  return col * 3 + row;
}

mat3_t mat3 = { 0, 1, 2,
                3, 4, 5,
                6, 7, 8 };

int offset = row_col_rm(1, 2); // offset 5
int value = mat[offset];       // value 6

int offset = row_col_cm(1, 2); // offset 7
int value = mat[offset];       // value 8
```

{% endraw %}

The bit where this can get a little confusing is if you want to start writing out matrices in your library (this is common for things like rotation and projection matrices). As the data is in a one dimensional array, and we're applying a convention that decides how we access the elements, when we write out a matrix in C/C++, it will always look like it's in row major format even if we're using the column major convention.

Access is the thing that makes the difference. When using row major, we walk each row on the left multiplying by each column on the right. If however we've decided to use the column major convention with this format, we need to update the traversal order.

One (not particularly ideal) solution is to keep the traversal the same, transpose both matrices, swap the order, multiply them and then transpose the result once more. This works, but it's a bunch of extra work. One fairly neat solution is to use our `row_col` lookup functions.

{% raw %}

```c++
// row major matrix multiply
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

// column major matrix multiply
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

{% endraw %}

The only difference is swapping out `row_col_rm` with `row_col_cm` and making sure we swap the order of the two matrices when we multiply them together. If we print the results of the `data` member we'll see we get the exact same result.

{% raw %}

```c++
// debug printing functions
for (int i = 0; i < 9; ++i) {
  std::cout << result.data[i] << ' ';
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
mat3_t result = a * b; // using row_col_rm version, pre-multiply

// prints
// 30 24 18 84 69 54 138 114 90 (memory order)
//
// 30 24 18
// 84 69 54
// 138 114 90
//
// display order in rows

// column major
mat3_t a = {1, 2, 3, 4, 5, 6, 7, 8, 9};
mat3_t b = {9, 8, 7, 6, 5, 4, 3, 2, 1};
mat3_t result = b * a; // using row_col_cm version, post-multiply

// prints
// 30 24 18 84 69 54 138 114 90 (memory order)
//
// 30 84 138
// 24 69 114
// 18 54 90
//
// display order in columns
```

{% endraw %}

One advantage to this approach is our basis vectors (e.g. axes and/or position) are contiguous in memory (we see `[x, y, z, x, y, z, x, y, z]` not [`x, x, x, y, y, y, z, z, z`]). This is often what graphics APIs such as OpenGL and DirectX expect when passing data to shaders.

#### Array Option #3

Interleave basis vectors and use column major.

{% raw %}

```c++
[0][3][6][1][4][7][2][5][8] // data

// conceptually becomes
[0][3][6]
[1][4][7]
[2][5][8]
```

{% endraw %}

In this approach the storage changes. We explicitly lay the data out in column major format but use the lookup and traversal we'd used when the storage was row major before.

One big advantage to this approach is it's possible to write column major matrices out in the source code.

{% raw %}

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

{% endraw %}

We also continue to use `row_col_rm` (or simply just `row_col`) as we're explicitly using the column storage order in how we layout the data. One downside to be aware of with this approach is the basis vectors are no longer contiguous in memory which might make certain access patterns slower (though this is likely negligible in practice) and you'll need to transpose the matrix before sending it to a graphics API (e.g. as a uniform) to ensure the data is contiguous and no longer interleaved.

{% raw %}

```c++
mat3_t a = {1, 4, 7,
            2, 5, 8,
            3, 6, 9};
mat3_t b = {9, 6, 3,
            8, 5, 2,
            7, 4, 1};
mat3_t result = b * a; // post-multiply

// prints
// 30 84 138 24 69 114 18 54 90 (memory order)
//
// 30 84 138
// 24 69 114
// 18 54 90
//
// display order in columns
```

{% endraw %}

__[Godbolt Link](https://gcc.godbolt.org/z/o3MGqjd9d)__

If we transpose `result` we get back the matrix in row major format

{% raw %}

```c++
mat3_t transpose(const mat3_t& mat) {
    return mat3_t{mat.data[0], mat.data[3], mat.data[6],
                  mat.data[1], mat.data[4], mat.data[7],
                  mat.data[2], mat.data[5], mat.data[8]};
}

mat3_t transposed = transpose(result);

// prints
// 30 24 18 84 69 54 138 114 90 (memory order)
//
// 30 24 18
// 84 69 54
// 138 114 90
//
// display order in rows
```

{% endraw %}

__[Godbolt Link](https://gcc.godbolt.org/z/xWGPWPanT)__

#### Multi-dimensional array

{% raw %}

```c++
struct mat3_t {
  float rows[3][3]; // name helps identify main array
};
```

{% endraw %}

Use a main array to traverse rows and a sub array to access column elements.

This approach might feel more intuitive than the single array method we touched on before. It does have some advantages but one thing to remember again is the main array usually represents the rows and the sub array represents the column elements. Of course it is possible to just interpret the main array as columns and the sub arrays as rows, but this is less common.

The memory layout will be identical to the single array approach, we just use the individual arrays to do lookups instead of using a `row_col` function.

{% raw %}

```c++
// row major matrix multiply
mat3_t operator*(const mat3_t& lhs, const mat3_t& rhs) {
    mat3_t m;
    for (int r = 0; r < 3; ++r) {
        for (int c = 0; c < 3; ++c) {
            float elem = 0.0f;
            for (int s = 0; s < 3; ++s) {
                elem += lhs.rows[r][s] * rhs.rows[s][c];
            }
            m.rows[r][c] = elem;
        }
    }
    return m;
}

              [row 0]    [row 1]    [row 2]
mat3_t a = {{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}};
mat3_t b = {{{9, 8, 7}, {6, 5, 4}, {3, 2, 1}}};
mat3_t result = a * b; // pre-multiply (row major)

// display storage order
for (int r = 0; r < 3; ++r) {
    for (int c = 0; c < 3; ++c) { // inner loop walks sub array
        std::cout << result.rows[r][c] << ' ';
    }
}

// display convention order
for (int r = 0; r < 3; ++r) {
    for (int c = 0; c < 3; ++c) {
        std::cout << result.rows[r][c] << " ";
    }
    std::cout << '\n';
}

// prints
// 30 24 18 84 69 54 138 114 90
//
// 30 24 18
// 84 69 54
// 138 114 90
//
// display order in rows
```

{% endraw %}

__[Godbolt Link](https://gcc.godbolt.org/z/bnr7Th1rr)__

To use column major order we can decide to make the main array columns instead of rows and update the traversal order slightly.

{% raw %}

```c++
struct mat3_t {
  float cols[3][3]; // name helps identify main array
};

// column major matrix multiply
mat3_t operator*(const mat3_t& lhs, const mat3_t& rhs) {
    mat3_t m;
    for (int r = 0; r < 3; ++r) {
        for (int c = 0; c < 3; ++c) {
            float elem = 0.0f;
            for (int s = 0; s < 3; ++s) {
                // notice column (c) and row (r) indices are swapped
                elem += lhs.cols[s][r] * rhs.cols[c][s];
            }
            m.cols[c][r] = elem;
        }
    }
    return m;
}

              [col 0]    [col 1]    [col 2]
mat3_t a = {{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}};
mat3_t b = {{{9, 8, 7}, {6, 5, 4}, {3, 2, 1}}};
mat3_t result = b * a; // post-multiply (column major)

// display storage order
for (int c = 0; c < 3; ++c) {
    for (int r = 0; r < 3; ++r) {  // inner loop walks sub array
        std::cout << result.cols[c][r] << ' ';
    }
}

// display convention order
for (int r = 0; r < 3; ++r) {
    for (int c = 0; c < 3; ++c) {
        std::cout << result.cols[c][r] << ' ';
    }
    std::cout << '\n';
}

// prints
// 30 24 18 84 69 54 138 114 90
//
// 30 84 138
// 24 69 114
// 18 54 90
//
// display order in columns
```

{% endraw %}

__[Godbolt Link](https://gcc.godbolt.org/z/8Mevd36xf)__

This will work but remember using columns as the first parameter when doing a lookup (`[col][row]` instead of `[row][col]`) is less common and may be confusing for users. Ensure to document this clearly in the API if this is something you decide to do.

For completeness you could also do the same as _Array Option 3_ and use rows as the main array and layout the data in column major format, using the same multiply implementation as for row major.

{% raw %}

```c++
struct mat3_t {
    float rows[3][3];
};

mat3_t a = {{{1, 4, 7},
             {2, 5, 8},
             {3, 6, 9}}};
mat3_t b = {{{9, 6, 3},
             {8, 5, 2},
             {7, 4, 1}}};

mat3_t result = b * a; // post-multiply

// prints
// 30 84 138 24 69 114 18 54 90 (memory order)
//
// 30 84 138
// 24 69 114
// 18 54 90
//
// display order in columns
```

{% endraw %}

__[Godbolt Link](https://gcc.godbolt.org/z/odvT93EcW)__

#### Composition of vector subtypes

The last approach to cover is one where we build a matrix out of vectors in our library. When we create these vectors we need to decide at the outset if they correspond to rows or columns (similar to the multidimensional array technique in the previous section).

Below is a brief example of what this might look like.

{% raw %}

```c++
struct vec3_t {
    float elems[3]; // using an array for ease of traversal
};

struct mat3_t {
    vec3_t rows[3];
};

// vector dot product, equivalent to matrix multiply innermost loop
float dot(const vec3_t& lhs, const vec3_t& rhs) {
    return lhs.elems[0] * rhs.elems[0] + lhs.elems[1] * rhs.elems[1] +
           lhs.elems[2] * rhs.elems[2];
}

mat3_t operator*(const mat3_t& lhs, const mat3_t& rhs) {
    mat3_t m;
    for (int r = 0; r < 3; ++r) {
        for (int c = 0; c < 3; ++c) {
            m.rows[r].elems[c] = dot( // build temporary column vector on rhs
                lhs.rows[r], vec3_t{rhs.rows[0].elems[c],
                                    rhs.rows[1].elems[c],
                                    rhs.rows[2].elems[c]});
        }
    }
    return m;
}

             [ row 0 ]  [ row 1 ]  [ row 2 ]
mat3_t a = {{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}};
mat3_t b = {{{9, 8, 7}, {6, 5, 4}, {3, 2, 1}}};
mat3_t result = a * b; // pre-multiply

for (int r = 0; r < 3; ++r) {
    for (int c = 0; c < 3; ++c) { // inner loop walks vector (columns)
        std::cout << result.rows[r].elems[c] << ' ';
    }
}

for (int r = 0; r < 3; ++r) {
    for (int c = 0; c < 3; ++c) {
        std::cout << result.rows[r].elems[c] << " ";
    }
    std::cout << '\n';
}

// prints
// 30 24 18 84 69 54 138 114 90
//
// 30 24 18
// 84 69 54
// 138 114 90
//
// display order in rows
```

{% endraw %}

__[Godbolt Link](https://gcc.godbolt.org/z/Kc8Yd46M7)__

If we'd prefer to use columns instead we can make a pretty small change to have this work.

{% raw %}

```c++
struct mat3_t {
    vec3_t cols[3];
};

mat3_t operator*(const mat3_t& lhs, const mat3_t& rhs) {
    mat3_t m;
    for (int r = 0; r < 3; ++r) {
        for (int c = 0; c < 3; ++c) {
            m.cols[c].elems[r] = dot( // build temporary row vector on lhs
                vec3_t{lhs.cols[0].elems[r],
                       lhs.cols[1].elems[r],
                       lhs.cols[2].elems[r]}, rhs.cols[c]);
        }
    }
    return m;
}

             [ col 0 ]  [ col 1 ]  [ col 2 ]
mat3_t a = {{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}};
mat3_t b = {{{9, 8, 7}, {6, 5, 4}, {3, 2, 1}}};
mat3_t result = b * a; // post-multiply

for (int c = 0; c < 3; ++c) {
    for (int r = 0; r < 3; ++r) { // inner loop walks vector (rows)
        std::cout << result.cols[c].elems[r] << ' ';
    }
}

for (int r = 0; r < 3; ++r) {
    for (int c = 0; c < 3; ++c) {
        std::cout << result.cols[c].elems[r] << " ";
    }
    std::cout << '\n';
}

// prints
// 30 24 18 84 69 54 138 114 90
//
// 30 84 138
// 24 69 114
// 18 54 90
//
// display order in columns
```

{% endraw %}

__[Godbolt Link](https://gcc.godbolt.org/z/vjzE8GoqP)__

Depending on which convention you pick (column major and post-multiply or row major and pre-multiply) keep in mind it'll be faster to return a matrix column if you're using column major as we don't have to construct a new object (we could if we wanted return a const reference to it). This means rows will be slightly more expensive to return (as there's a small amount of work to build one) and we can't return a reference to this type as it is constructed on the fly and returned by value (and vice versa when dealing with row major).

{% raw %}

```c++
// possible API
struct mat3_t {
    vec3_t col(int i) const { return cols[i]; } // trivial
    vec3_t row(int i) const { return vec3_t{cols[0].elems[i], // non-trivial
                                            cols[1].elems[i],
                                            cols[2].elems[i]}; }
private:
    vec3_t cols[3]; // actual implementation hidden
};
```

{% endraw %}

If we're returning a vector (column or row) by value we also need to make it obvious to the caller that modifying this type will not change the underlying matrix (this can be a gotcha and is one reason to prefer the array based approach).

In the example above we used an array in the vector type to make traversal simpler. Most libraries use a `union` to allow access through `float x, y, z;` as well as via an array. This is possible in C however it's not _technically_ allowed in C++ (see [this Stack Overflow post](https://stackoverflow.com/a/11996970/1947066) for more details). A different solution is to simply write accessor member functions (something like `get_x(), set_x(...)` etc...) or provide operator overloads for the `[]` operator on your type. A safe technique to provide array access to data members does exist (I first learned about this technique [here](https://www.gamedev.net/forums/topic/328530-c-union-question/3126294/?page=1) on GameDev.net and wrote a little more about it [here](https://github.com/pr0g/as#specializations) in a math library I implemented) but it is a little complicated to get your head around initially.

This approach of using a subtype can also be very helpful if using a SIMD implementation (that way operations can be composed) but the specifics and implementation details of SIMD are outside the scope of this post.

### Conclusion

I hope that just about covers the main approaches you might want to consider when implementing a math library for games. There's no 100% right answer and each approach comes with pros and cons (e.g. flexibility vs simplicity).

If you'd like to explore more you can check out this math library I created written in C++.

- [as - almost something](https://github.com/pr0g/as).

The library uses row major storage/layout but allows either row or column major convention to be used. It is selected by defining either `AS_COL_MAJOR` or `AS_ROW_MAJOR`.

I have another math library witten purely in C (no templates or operator overloading).

- [as-c-math](https://github.com/pr0g/as-c-math).

It uses column major storage/layout and convention. It isn't as mature or fully featured as [as](https://github.com/pr0g/as) but it's slowly growing and improving. I hope they might prove useful to review to learn more about what was covered.
