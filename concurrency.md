# Intro
Std lib support is aimed to support systems-level concurrency, not a higher level concurrency model. Can concurrently exec multiple threads in a single address space and for that C++ provides: memory model and set of atomic ops (allow lock-free programmin). Ths memory model ensures that if I as a programmer can avoid data races, then everything would work as expected. 

Main supports: thred, mutex, lock(), packaged_task, future. These do not incur perfromance penalties since built directly over what OS offers, hence don't also gurantee and significant perfromance improvement over what OS offers.

Alternative to explicit consurrency: Use a parallel algo to leverage multoiple execution engines for better performance.

# 15.2 Task and thread
Task: a computation that can be executed concurrently with other computations. It is a `function` or `function object`

Thread: System level representation of Task in a program i.e. to exec a task, launch it using `std::thread` with the task as its arg.
```c++
void f();

struct F {
    void operator()(); // F's call operator
};

void user() {
    thread t1{f};   // f() execs in separate thread
    thread t2{F()}; // F()() execs in separate thread

    t1.join();
    t2.join(); // join() ensure we dont exit user() until threads have completed ie. wait for thread to terminate
}
```
Threads of 1 program share 1 single address space (different from process as process doesn't share data). Since shared address speace => can communicate thru shared data objects. This comm needs to be ctrlled by locks etc to prevent data race (unctrlled concurrent access to a var).

```c++
// tricky to program concurrent tasks
void f() {
    cout << "hello";
}

struct F {
    void operator()() {
        cout << "prallel wld";
    }
};
// both f and F use cout withut any sync => resulting output will be unpredicatble
// only a sepcific guarantee in the std saves a data race within `ostream` that could cuase a crash
```
Aim: Keep tasks completely separate except when the comm in: simple and obvious ways. Pass args, get result back and ensure there is not use of shared data in between.

## 15.3 arg passing
```C++
void f(vector<double>&& v);

struct F {
    vector<double>& v;

    F(vector<double>& vv) : v(vv) {};

    void operator()();
};

int main() {
    vector<double> vec1{1,2,3,4,5};
    vector<double> vec2{6,7,8,9,10};
    // ref() is a type fn from <functional> that tells the variadic template to treat vec1 as a ref vs as object, else vec1 will pass by val
    thread t1{f, ref(vec1)}; // thread variadic template ctor 
    thread t2{F{vec2}}; // F{vec2} is saving ref to vec2 in its `v` but passing vec2 by val would reduce risk of no other task changnig vec2 meanwhile

    t1.join();
    t2.join();
}
```
Here args are passed by non const ref only becuase we assume tasks will modify the val of data refreed, it is sneaky way to return a result.
## 15.4 return result
Another way to get results over non-const ref arg pass is to pass the arg as a const ref and another arg for the lcoation of the place to store results.
```C++
void f(const vector<double>& v, double* res); // in: v, out: *res

class F{
public:
    F(const vector<double>& vv, double* p): v{vv}, res{p} {}

    void operator()();
private:
    const vector<double>& v; // in
    double* res; // out
};

double g(const vecotr<double>& ); // use the return val

void user(vector<double>& vec1, vector<double> vec2, vector<double> vec3) {
    double res1, res2, res3;

    thread t1 {f, cref(vec1), &res1};
    thread t2 {F{vec2, &res2}};
    thread t3 { [&](){                  // capture local vars by ref
                        res3 = g(vec3);
                     }};
    t1.join();
    t2.join();
    t3.join();
    cout << "res 1" << res1; // ....
}
```
## 15.5 Sharing data

