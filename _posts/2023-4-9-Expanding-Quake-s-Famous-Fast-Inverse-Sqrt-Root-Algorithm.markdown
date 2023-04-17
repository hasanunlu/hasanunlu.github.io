---
layout: post
title:  "Expanding Quake's Famous Fast Inverse Square Root Algorithm"
date:   2023-4-9 10:46:39 -0700
# categories: jekyll update
---

In 1999, an unintuitive method was employed in the Quake III Arena graphics library to quickly calculate surface normals for lighting and shading. The function calculating inverse square root is in below. The original code for the repository can be found [here](https://archive.softwareheritage.org/browse/content/sha1_git:bb0faf6919fc60636b2696f32ec9b3c2adb247fe/?origin_url=https://github.com/id-Software/Quake-III-Arena&path=code/game/q_math.c&revision=dbe4ddb10315479fc00086f08e25d968b4b43c49&snapshot=4ab9bcef131aaf449a7c01370aff8c91dcecbf5f#L549-L572).

```c
float Q_rsqrt( float number )
{
	long i;
	float x2, y;
	const float threehalfs = 1.5F;

	x2 = number * 0.5F;
	y  = number;
	i  = * ( long * ) &y;                       // evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck? 
	y  = * ( float * ) &i;
	y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//	y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed

	return y;
}
```

The advantage stems from the way numbers are stored in floating-point format like $x=\pm(1+m_x)2^{e_x}$, which allows power operations to be converted into addition and subtraction in the exponent field. Also $log_2(1+x)\approx x$ in the mantissa field. This assists in finding an initial approximation, with one or two Newton-Raphson iterations yielding highly accurate results for compute heavy functions, such as the inverse square root.
\
\
![IEEE 754 Single-Precision Format](/assets/ieee754_float.png){:style="display:block; margin-left:auto; margin-right:auto"}
<div align="center">
IEEE 754 Single-Precision Format
</div>
\
\
In this article, I will derive a generic formula and code to apply this method for computing $x^{\frac{1}{m}}$ where $m \in Z$ and $m \neq 0$.
\
\
**First Step: Deriving Magic Number and Initial Guess**
\
Let's represent the floating-point number in its integer-casted form. For the time being, assume all values are positive, which means the sign bit is zero.
\
\
$I_x = E_x \cdot L + M_x$ where $L=2^{23}$ for single precision floating point format.
\
\
The floating point number's value representaion: 
\
\
$x = (1 + \frac{M_x}{L})\cdot 2^{E_x - B}$ where $B=127$ for single precision floating point format.
\
\
Now we need to represent $I_x$ in terms of $x$. However dealing with $x$ in this form is not easy. Let's take $log_2$ of both side.
\
\
$log_2 x = log_2 (1 + \frac{M_x}{L}) + E_x - B$, we know that $\frac{M_x}{L} < 1$.
\
\
$log_2 x \approx \frac{M_x}{L} + \sigma + E_x - B$ where $\sigma$ is such constant to improve approximation accuracy. In the step three, we will calculate $\sigma$. Please also check $log(1+x) \approx x$ for near zero values.
\
\
$log_2 x \approx  \frac{M_x}{L}  + \sigma + E_x - B$, subsitue $M_x$ with $I_x -  E_x \cdot L$
\
\
$log_2 x \approx \frac{I_x -  E_x \cdot L}{L}  + \sigma + E_x - B$
\
\
$log_2 x \approx \frac{I_x}{L} - E_x  + \sigma + E_x - B$
\
\
$log_2 x \approx \frac{I_x}{L} + \sigma - B$
\
\
The value we are trying to find is $y = x^{\frac{1}{m}}$.
\
\
$log_2 y \approx \frac{I_y}{L} + \sigma - B$
\
\
At the same time, $log_2 y = log_2 x^{\frac{1}{m}}$
\
\
$log_2 y = \frac{1}{m}log_2 x$
\
\
$\frac{I_y}{L} + \sigma - B \approx \frac{1}{m}(\frac{I_x}{L} + \sigma - B)$
\
\
$\frac{I_y}{L} \approx \frac{I_x}{m\cdot L} + (\frac{1}{m} - 1)(\sigma - B)$
\
\
$I_y \approx \frac{I_x}{m} + L(\frac{1}{m} - 1)(\sigma - B)$
\
\
$I_y \approx \frac{I_x}{m} + (1 - \frac{1}{m})L(B - \sigma)$
\
\
In here $(1 - \frac{1}{m})L(B - \sigma)$ corresponds magic number where $L=2^{23}$ and $B=127$ for IEEE 754 single precision format.
\
\
**Second Step: Newton-Raphson Method**
\
We need to find the $1/m$ power of a number, so if we define this as an unknown, the problem can be formulated as follows:
\
\
$f(x) = x^{m} - a$ to find a root of this function which is $a^{\frac{1}{m}}$
\
\
The Newthon-Raphson iteration for this $f(x)$ would be
\
\
$x_{n+1} = x_n - \frac{f(x_n)}{f'(x_n)}$
\
\
$x_{n+1} = x_n - \frac{x_n^m - a}{m\cdot x_n^{m-1}}$
\
\
$x_{n+1} = x_n - \frac{1}{m}(x_n - a \cdot x_n^{1-m})$
\
\
$x_{n+1} = \frac{(m - 1) \cdot x_n + a \cdot x_n^{1-m}}{m}$
\
\
$x_{n+1} = \frac{x_n \cdot ( m - 1 + a \cdot x_n^{-m})}{m}$
\
\
**Third Step: Finding $\sigma$**
\
There are several approaches to finding it. In this case, we will focus on minimizing the maximum error for any given value within the $[0, 1]$ interval. The error function can be defined as $|log_2 (x+1) - (x+\sigma)|$. This function has one local maximum and two interval edge points. The error amounts at the interval boundaries (at $0$ and $1$) are both equal to $\sigma$.
\
\
![Error Graph](/assets/error_graph.png){:style="display:block; margin-left:auto; margin-right:auto"}
<div align="center">
Logx, x and Relative Error Graph
</div>
\
Check Desmos graphing calculator [here](https://www.desmos.com/calculator/gizsi2hjz3) to edit and play with approxmation variable.
\
\
We need to get the derivative to find local maxima.
\
\
$\frac{\partial error(x)}{\partial x}=\frac{1}{ln(2)*(x+1)} - 1 = 0$
\
\
$x=1 - \frac{1}{ln(2)} = 0.44269504088$
\
\
The error value at this local maxima should not exceed the error amounts at the boundaries.
\
\
$log_2 (x+1) - (x+\sigma)=\sigma$ where $x=0.44269504088$.
\
\
$log_2(0.44269504088 + 1) - 0.44269504088 = 2\sigma$
\
\
$\sigma = (log_2(1.44269504088) - 0.44269504088)/2$
\
\
$\sigma = 0.04303566602$
\
\
Here is the generic code for taking power of $1/m$. You can modify and optimize it for specific $m$ values as needed.

```c++
#include <iostream>
#include <cmath>

using namespace std;

class Fast_1_M_Math
{
    private:
    int magic_number;
    int m;

    float integer_power(float x, int power)
    {
        float result = 1.f;

        while (power)
        {
            if (power & 1)
            {
                result = result * x;
            }

            power = power >> 1;

            if (!power)
            {
                break;
            }

            x = x * x;
        }

        return result;
    }

    public:

    Fast_1_M_Math(int m_number) {
        const float sigma = 0.04303566602; // Approximation constant
        const float B = 127.f; // IEEE-754 single precision exponent offset
        const float L = 1 << 23; // IEEE-754 single prrecision mantissa length
        m = m_number;
        float partial = (1.f - 1 / (float)(m_number)) * (B - sigma) * L;
    	magic_number = (int)(partial);
    }

    float get(float a)
    {
       	if (m == 0 || ((m & 1) == 0 && a < 0.f))
       	{
        	return NAN;
       	}

       	int i = *((int *)&a);
       	i = i / m + magic_number;
       	float x_n = *((float *)&i);

    	if (m > 0)
    	{
            // One iteration Newton-Raphson
            return x_n * (m - 1 + a / integer_power(x_n, m)) / m; 
    	}
       	else
       	{
            // One iteration Newton-Raphson
            return x_n * (m - 1 + a * integer_power(x_n, -m)) / m;
       	}
    }
};

int main()
{
    Fast_1_M_Math inverse_square(-2);

    float x = 1123.4231231;
    float a1 = inverse_square.get(x);
    float a2 =  1.f / sqrt(x);
    cout << "fast: " << a1 << endl;
    cout << "lib: " << a2 << endl;

    return 0;
}
```
\
**References**
- Lomont, Chris (February 2003). "Fast Inverse Square Root" (PDF). Retrieved 2009-02-13.
- https://en.wikipedia.org/wiki/Fast_inverse_square_root

