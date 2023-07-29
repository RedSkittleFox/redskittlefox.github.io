---
layout: post
title: Designing a C++ Thread-Pool
---

The goal of this blog post is to explore different design aspects of a thread-pool from both the performance and API point of view. This article will explore different task-scheduling policies, different queue types - atomic and lock-based, and different type-erasure methods to store tasks (callable objects). This article will also test the effects of using aligned memory.

# Experimental Thread Pool Design Considerations and API
Our thread-pool is composed of three pieces:

1. Task object - type-erased callable that when invoked set's future's value.
2. Task queue - container responsible for inter-thread synchronization and task storage.
3. Thread pool - object responsible for thread management and providing a public API to store tasks.

Additionally memory management should be taken into account, if possible, fixed size queues should used and invokable objects should be placed in memory footprints of the task object. 

Because we want to benchmark different configurations we will first design "experimental" thread pool api that allows us to plug in different queues and task types together and configure different task-scheduling policies and synchronization mechanisms.

## Task object API

Let's first figure out what we need. Our task must behave somewhat like an `std::function`, except this time, we perform complete type erasure, we store passed arguments by values (or reference by curtesy of `std::ref`), the `invoke()` method will update earlier acquired `std::future`. We need to be able to, regardless of stored callable, construct the task in place, move it, invoke it and destroy it. To this we will divide our API into two objects : task-type - type erased object with `move_to(void*)`, `destroy_this()` and `execute()` methods. The second object will be task-trait - contains minimal memory footprint required to store a task, aforementioned task-type and a `construct_task_at` function. This is the API:

```cpp
template<size_t MaxFootprint>
struct X_task_impl
{
    void execute();
	void move_to(void* to);
    void destroy_this();
};

struct X_task
{
    // Minimal memory footprint size required to store the task
    inline static constexpr std::size_t minimal_task_size = ... ;

    template<size_t MaxFootprint>
	using task_type = X_task_impl<MaxFootprint> ;

	template<
        size_t MaxFootprint, // Size of location buffer
        class Job, // Callable
        class... Args, // Passed arguments
		class invoke_result_t = std::invoke_result_t<std::unwrap_ref_decay_t<Job>, std::unwrap_ref_decay_t<Args>...>>
	static void construct_task_at(std::promise<invoke_result_t>&& p, task_type<MaxFootprint>* location, Job&& job, Args&&... args);
};
```

One might ask, why don't we just make these a static_members of the `task_type`. This is so we can pass potential generic task-types (parameterized by footprint size) as template arguments. One could ask, why not just make `X_task` generic itself and pass it as template argument? Well, I wanted to be able to parameterize everything with one template argument of `thread_pool` like so:
`thread_pool<task_type, buffer_type, lock_type, MaxTaskFootprint = 64, JobDistribution>` rather than `thread_pool<task_type<64>, buffer_type<64>, lock_type, MaxTaskFootprint = 64, JobDistribution>`. 

We shouldn't worry about this too much though, the modularity will not be present in the final *production* version of the thread pool.

## Buffer object API

For the sake of simplicity, all our buffers will be of fixed capacity - no dynamic reallocations. The main objective of the buffer objects is to enable efficient and thread safe task storage, enqueueing and dequeueing. To ensure thread safety we can either provide an atomic-based queue or employ lock-based solution. Our experimental buffer API will have the following interface:
```cpp
template<std::size_t MaxFootprint, task_like Task, lock_like Lock>
struct X_buffer_impl
{
    X_buffer_impl(std::size_t capacity); 

    bool try_pop(task_type* dest);

    template<class Job, class... Args,
        class invoke_result_t = std::invoke_result_t<std::unwrap_ref_decay_t<Job>, std::unwrap_ref_decay_t<Args>...>>
    bool try_push(std::promise<invoke_result_type>&& promise, Job&& job, Args&&... args);

    std::size_t size() const noexcept;
    std::size_t capacity() const noexcept;
    bool empty() const noexcept;

    void notify_one();
    void notify_all();

    template<std::predicate Predicate>
    bool pop_or_wait(task_type* dest, Predicate stop_waiting);
}

struct X_buffer
{
	template<size_t MaxFootprint, task_like Task, lock_like Lock>
	using type = X_buffer_impl<MaxFootprint, Task, Lock>;

    using is_atomic = std::bool_constant<...>;
};
``` 

Let's look closer at the API now. All of the methods (except constructor and destructor) must be thread safe. Method `try_pop(task_type*)` will try pop the task into the memory location specified (move it there). Analogically `try_push` will emplace new task into the queue and update the promise on it's completition.

In case of `std::mutex` based locking scheme, we would like to take full advantage of `std::condition_variable`, to this end we provide `notify_one()`, `notify_all()` and `pop_or_wait()`. First two methods will wake up sleeping thread, the last one will enter a conditional-variable sleep if the pop fails. Not every queue has to implement sleeping mechanism, if it doesn't, `pop_or_wait` call can just be forwarded to `try_pop`.

To reduce code duplications [`buffer_lock_adapter`](https://github.com/RedSkittleFox/thread_pool/blob/5e999baf7feb2ea8cec9aa5ef394a80188711554/src/fox_thread_pool/implementation/buffer_lock_adapter.hpp#L11C27-L11C27) was implemented that automatically provides lock-basked implementation for non-atomic queues. Atomic queues can use [`ATOMIC_LOCKABLE_BUFFER_IMPL`](https://github.com/RedSkittleFox/thread_pool/blob/5e999baf7feb2ea8cec9aa5ef394a80188711554/src/fox_thread_pool/implementation/buffer_lock_adapter.hpp#L106) macro to generate default empty implementations.

## The thread-pool API
Finally, lets focus on the simplified thread-pool API. Don't worry, we are going to extend it in the final implementation. We want to be able to control job distribution policy, task type, buffer type, lock type and footprint size. Thread pool should provide API to schedule tasks and work whilst waiting on waitable object (`std::future` for now).
```cpp
enum class job_distribution_policy
{
	single_job_queue,
	round_robin,
	distributed
};

template<
    implementation::task_like Task,
    implementation::buffer_like JobQueueType,
    class Lock,
    std::size_t MaxTaskFootprint,
    job_distribution_policy JobDistributionPolicy
>
class thread_pool
{
public:
    thread_pool(std::size_t num_threads, size_t queue_capacity);

    thread_pool(thread_pool&&) noexcept;

    thread_pool& operator=(thread_pool&&) noexcept;

public:
    template<class T>
    void work_and_wait(std::future<T>& f);

    template<class Job, class... Args,
        class invoke_result_t = std::invoke_result_t<std::unwrap_ref_decay_t<Job>, std::unwrap_ref_decay_t<Args>...> 
    >
    auto schedule(Job&& job, Args&&... args) -> std::future<invoke_result_t>;
}
```

We will implement three job policies with a little twist.
1. Single job queue - There is a single queue shared among all the threads.
2. Round Robin - Each queue has it's own job queue, when one queue runs out of jobs, it goes to another one and steals them. Jobs are inserted to subsequent queues.
3. Distributed - Each queue has it's own job queue however this time jobs are inserted to the queue with the least amount of work left. Additionally, queue is lock-based, size will be cached inside the thread-pool using relaxed atomics.

# Experimental Thread Pool Implementation

This section will go over implementation details of the experimental thread pool. Some aspects will be discussed in more detail whilst others will be just briefly mentioned. If you would like to explore those aspects in detail, feel free to take a look at the [`source code`](https://github.com/RedSkittleFox/thread_pool).

## Implementing functional task
This section will focus on the most basic and naive implementation of the task object - to perform type erasure we will use `std::move_only_function` and a lambda. This implementation is very simple but also very limiting, we have no control over the memory.

```cpp
class functional_task_impl
{
	std::move_only_function<void()> function_;
public:
	// ... default the big 5 ...

	template<class Job, class... Args,
		class invoke_result_t = std::invoke_result_t<std::unwrap_ref_decay_t<Job>, std::unwrap_ref_decay_t<Args>...>>
	functional_task_impl(std::promise<invoke_result_t>&& promise, Job&& job, Args&&... args)
	{
		static_assert(std::is_move_constructible_v<Job>, "[Job] is not move constructible.");
		static_assert(std::conjunction_v<std::is_move_constructible<Args>...>, "[Args] are not move constructible.");

		function_ = [
			p = std::move(promise),
            // Perform copy-decay
			j = static_cast<std::decay_t<Job>>(std::forward<Job>(job)),
			tpl = std::make_tuple(std::forward<Args>(args)...)
		]() mutable -> void {
			try
			{
                // If function return's void, we cannot initialize it with a value
				if constexpr (std::is_same_v<invoke_result_t, void>)
				{
					std::apply(j, tpl);
					p.set_value();
				}
				else
				{
					p.set_value(std::apply(j, tpl));
				}
			}
			catch (...)
			{
                // Forward any exception
				p.set_exception(std::current_exception());
			}
		};
	}

public:
	void execute()
	{
		function_();
	}

	void move_to(void* to)
	{
		static_assert(std::is_move_constructible_v<decltype(*this)>);
        // Move construct this at memory location
		std::construct_at(static_cast<functional_task_impl*>(to), std::move(*this));
	}

	void destroy_this()
	{
        // Call destructor on this
		std::destroy_at(this);
	}
};

struct functional_task
{
	inline static constexpr std::size_t minimal_task_size = sizeof(functional_task_impl);

	template<size_t> using task_type = functional_task_impl;

	template<size_t MaxFootprint, class Job, class... Args,
		class invoke_result_t = std::invoke_result_t<std::unwrap_ref_decay_t<Job>, std::unwrap_ref_decay_t<Args>...>>
	static void construct_task_at(
		std::promise<invoke_result_t>&& p, task_type<MaxFootprint>* location, Job&& job, Args&&... args)
	{
	    static_assert(MaxFootprint >= sizeof(functional_task_impl), "Cannot place object in footprint.");

        // Perform in-place construction
		std::construct_at(reinterpret_cast<functional_task_impl*>(location),
		    std::move(p), std::forward<Job>(job), std::forward<Args>(args)...);
	}
};
```

As you can see, our type erasure works by capturing the promise, job and parameters and then calling them from the lambda body, which can be stored using `std::move_only_function<void()>`. In order to ensure that the job object and arguments are available during deferred execution of the job, we need to capture them by value (or if needed, explicitly use `std::ref`). To do this, we perform *copy-decay* operation on the job during capture `j = static_cast<std::decay_t<Job>>(std::forward<Job>(job))` and then we use `std::make_tuple` to *copy-decay* parameters.

Complete implementation can be found [here](https://github.com/RedSkittleFox/thread_pool/blob/main/src/fox_thread_pool/implementation/task/task_functional.hpp).

## Implementing virtual-dispatch based task
As one might suspect, this implementation will use virtual inheritance to handle type erasure. We will start by defining our base class.
```cpp
class virtual_task_base
{
public:
	// ... big 5 ...	
	virtual ~virtual_task_base() noexcept = default;
	virtual void execute() = 0;
	virtual void move_to(void* to) = 0;
	virtual void destroy_this() = 0;
};
```
We will provide two concrete implementations, one will store the callable in it's footprint if the size allows it. Another one will store the concrete implementation no. 1 inside the `unique_ptr``. This way we can store jobs bigger than provided memory footprints at the cost of dynamic memory allocations. 

```cpp
template<size_t Version, class Job, class... Args>
class virtual_task_impl : public virtual_task_base
{
	using this_type = virtual_task_impl<0, Job, Args...>;
	using invoke_result_t = std::invoke_result_t<Job, Args...>;

    // We store variables and job in the memory footprint of the task
	Job job_;
	[[no_unique_address]] std::tuple<std::unwrap_ref_decay_t<Args>...> args_;
	std::promise<invoke_result_t> promise_;
public:
    // ... big 5 ...	

	template<class UJob, class... UArgs, // Allow forwarding
		class invoke_result_r = std::invoke_result_t<std::unwrap_ref_decay_t<Job>, std::unwrap_ref_decay_t<Args>...>>
	virtual_task_impl(std::promise<invoke_result_r>&& p, UJob&& job, UArgs&&... args)
	requires
		std::is_constructible_v<std::unwrap_ref_decay_t<Job>, UJob> && 
		std::is_constructible_v<std::tuple<std::unwrap_ref_decay_t<Args>...>, UArgs...>
		: job_(std::forward<UJob>(job)), args_(std::make_tuple(std::forward<UArgs>(args)...)), promise_(std::move(p))
	{
		static_assert(std::is_move_constructible_v<this_type>, "Task type with [Job] and [Args] is not move constructible.");
	}

public:
	void execute() override
	{
		try
		{
			if constexpr (std::is_same_v<invoke_result_t, void>)
			{
				std::apply(job_, args_);
				promise_.set_value();
			}
			else
			{
				promise_.set_value(std::apply(job_, args_));
			}
		}
		catch (...)
		{
			// If exception occurred, store it in promise 
			promise_.set_exception(std::current_exception());
		}
	}

	void move_to(void* to) override
	{
		static_assert(std::is_move_constructible_v<decltype(*this)>);
		std::construct_at(static_cast<this_type*>(to), std::move(*this));
	}

	void destroy_this() override
	{
		std::destroy_at(this);
	}
};
```

Notice how this time, the execute function contains the implementation instead of the lambda. We perform our type erasure via virtual functions. Now it should be clear why we created two special functions `move_to` and `destroy_this`. It makes it possible to use virtual types with without dealing with dynamic memory or [object slicing](https://en.wikipedia.org/wiki/Object_slicing). Any call to the via the base interface will in-fact invoke one of these type-specialized methods.

Let's take the look at the second concrete implementation, which uses `unique_ptr` to store the data.

```cpp
template<class Job, class... Args>
class virtual_task_impl<1, Job, Args...> : public virtual_task_base
{
	using this_type = virtual_task_impl<1, Job, Args...>;
	using invoke_result_t = std::invoke_result_t<Job, Args...>;

	std::unique_ptr<virtual_task_impl<0, Job, Args...>> task_;
	
public:
    // ... big 5 ...	

	template<class UJob, class... UArgs,
		class invoke_result_t = std::invoke_result_t<std::unwrap_ref_decay_t<Job>, std::unwrap_ref_decay_t<Args>...>>
	requires
		std::is_constructible_v<typename decltype(task_)::element_type, std::promise<invoke_result_t>&&, UJob, UArgs...>
	virtual_task_impl(std::promise<invoke_result_t>&& p, UJob&& job, UArgs&&... args)
		: task_(std::make_unique<typename decltype(task_)::element_type>(std::move(p), std::forward<UJob>(job), std::forward<UArgs>(args)...))
	{
		assert(task_ != nullptr);
	}

public:
	void execute() override
	{
		task_->execute();
	}

	void move_to(void* to) override
	{
		static_assert(std::is_move_constructible_v<decltype(*this)>);
		std::construct_at(static_cast<this_type*>(to), std::move(*this));
	}

	void destroy_this() override
	{
		std::destroy_at(this); // We can safely call destructor here without 
        // task_->destroy_this(), we are storing a concrete type
	}
};
```

As demonstrated, this implementation just stores the job in the heap allocated memory. Now the final part of the puzzle is to select the implementation. If job is movable and can be placed in the memory footprint, use implementation 0, otherwise use implementation 1.

```cpp
struct virtual_task
{
	template<size_t>
	using task_type = virtual_task_base;

	template<class Job, class... Args>
	inline static constexpr std::size_t _task_size = sizeof(virtual_task_impl<0, Job, Args...>);

	inline static constexpr std::size_t minimal_task_size = sizeof(virtual_task_impl<1, void(*)()>);

	template<size_t MaxFootprint, class Job, class... Args,
		class invoke_result_t = std::invoke_result_t<std::unwrap_ref_decay_t<Job>, std::unwrap_ref_decay_t<Args>...>>
	static void construct_task_at(std::promise<invoke_result_t>&& p, task_type<MaxFootprint>* location, Job&& job, Args&&... args)
	{
		using concrete_task_type_t
			= std::conditional_t <
			std::is_move_constructible_v<virtual_task_impl<0, std::decay_t<Job>, std::decay_t<Args>...>> && // Is movable
			_task_size<std::decay_t<Job>, std::decay_t<Args>...> <= MaxFootprint, // Can be placed locally in the queue
			virtual_task_impl<0, std::decay_t<Job>, std::decay_t<Args>... > ,
			virtual_task_impl<1, std::decay_t<Job>, std::decay_t<Args>...>
			>;

		static_assert(MaxFootprint >= sizeof(concrete_task_type_t), "Cannot place object in footprint.");

		std::construct_at(reinterpret_cast<concrete_task_type_t*>(location),
			std::move(p), std::forward<Job>(job), std::forward<Args>(args)...);
	}
};
```

Complete implementation can be found [here](https://github.com/RedSkittleFox/thread_pool/blob/main/src/fox_thread_pool/implementation/task/task_virtual.hpp).

## Implementing function pointer based task
This implementation will perform type-erasure by using function pointers instead of virtual dispatch. In essence, we will create our own vtable. This time we will consider two approaches though. First approach will use three different function pointers to store pointers to our proxy functions - it will be a bit more efficient but takes up more storage, second approach will use a jump-table with a single function pointer. 

Handling and creation of proxy-templated functions will be handled by the base class. The derived class is the same as as in case of virtual-class, except this time we call a special base constructor that will generate templated methods.

```cpp
template<class Base, size_t Version, class Job, class... Args>
class fptr_task_impl : public Base // we can select one of two bases
{
	...

public:
	template<class UJob, class... UArgs>
	requires ...
	fptr_task_impl(std::promise<invoke_result_t>&& p, UJob&& job, UArgs&&... args)
		:
		Base(std::type_identity<this_type>{}), // Base constructor invoked here
		...
	{}

	...
}
```

Now let's take a look at what the base class does for us.

```cpp
class fptr_task_0_base
{
	void(*execute_)(void* p_this);
	void(*move_to_)(void* p_this, void* dest);
	void(*destroy_this_)(void* p_this);

private:
	template<class T>
	static void execute_proxy(void* p_this)
	{
		static_cast<T*>(p_this)->execute();
	}

	template<class T>
	static void move_to_proxy(void* to, void* p_this)
	{
		static_cast<T*>(p_this)->move_to(to);
	}

	template<class T>
	static void destroy_this_proxy(void* p_this) 
	{
		static_cast<T*>(p_this)->destroy_this();
	}

public:
	// ... big five ...

	template<class T>
	fptr_task_0_base(std::type_identity<T>)
		:
		execute_(&fptr_task_0_base::execute_proxy<T>),
		move_to_(&fptr_task_0_base::move_to_proxy<T>),
		destroy_this_(&fptr_task_0_base::destroy_this_proxy<T>) {}

public:
	void execute() 
	{
		execute_(this);
	}

	void move_to(void* to) 
	{
		move_to_(to, this);
	}

	void destroy_this() 
	{
		destroy_this_(this);
	}
};
```

As you can see, this implementation stores 3 function pointers and then invokes them in the respective methods. The templated constructor instantiates templated proxy functions based on the type passed to it. We deduce the type by using a dummy `std::type_identity<T>` type since we cannot explicitly specify constructor's template arguments. 

The second implementation will use a jump table and a single proxy function instead of three.

```cpp
class fptr_task_1_base
{
	void(*util_function_)(void* p_this, void* param, uint8_t mode);

private:
	template<class T>
	static void util_function_proxy(void* p_this, void* param, uint8_t mode)
	{
		switch (mode)
		{
		case 0:
			return static_cast<T*>(p_this)->execute();
		case 1:
			return static_cast<T*>(p_this)->move_to(param);
		case 2:
			return static_cast<T*>(p_this)->destroy_this();
		default:
			std::unreachable();
		}
	}

public:
	...

	void execute()
	{
		util_function_(this, nullptr, 0);
	}

	void move_to(void* to)
	{
		util_function_(this, to, 1);
	}

	void destroy_this()
	{
		util_function_(this, nullptr, 2);
	}
}
```

The task-trait is analogous to the virtual task's task-trait.

Complete implementation can be found [here](https://github.com/RedSkittleFox/thread_pool/blob/main/src/fox_thread_pool/implementation/task/task_fptr.hpp).

## Benchmarking task implementations
To benchmark the tasks I will use this simple benchmark. It will perform all the operations we would expect the task object to have during it's lifetime.

```cpp
template<fox::implementation::task_like Task, bool Big>
static void task_benchmark(benchmark::State& state)
{
    auto task = [&]
    {
        if constexpr (Big)
            return [v = std::array<uint8_t, 80>()](bool b) -> bool {return !b; };
        else
            return static_cast<bool(*)(bool)>([](bool b) -> bool { return !b; });
    }();

    static constexpr size_t footprint = 64;
    std::array<uint8_t, footprint> task_memory_0;
    std::array<uint8_t, footprint> task_memory_1;

    using task_t = typename Task::template task_type<footprint >;
    task_t* task_0 = reinterpret_cast<task_t*>(std::data(task_memory_0));
    task_t* task_1 = reinterpret_cast<task_t*>(std::data(task_memory_1));

    for (auto _ : state)
    {
        {
            state.PauseTiming();
            std::promise<bool> promise;
            std::future<bool> f = promise.get_future();
            state.ResumeTiming();

            // Construct task in memory 1
            Task::construct_task_at<footprint>(std::move(promise), task_0, task, true);

            task_0 = std::launder(task_0);

            task_0->move_to(task_1);
            task_0->destroy_this();

            task_1 = std::launder(task_1);

            task_1->execute();
            task_1->destroy_this();

            state.PauseTiming(); // Destroy promise and future
        }
        state.ResumeTiming();
    }
}
```
These are the result, as you can probably tell, false means that tasks fits in the footprint, true means it uses dynamic allocation. Performance results reflect that.
```
-------------------------------------------------------------------------------------
Benchmark                                           Time             CPU   Iterations
-------------------------------------------------------------------------------------
task_benchmark<fi::virtual_task, false>           387 ns          311 ns      2508800
task_benchmark<fi::virtual_task, true>            455 ns          486 ns      1672533
task_benchmark<fi::functional_task, false>        398 ns          374 ns      1672533
task_benchmark<fi::functional_task, true>         431 ns          366 ns      1493333
task_benchmark<fi::fptr_task_0, false>            370 ns          324 ns      2800000
task_benchmark<fi::fptr_task_0, true>             437 ns          352 ns      1730207
task_benchmark<fi::fptr_task_1, false>            386 ns          397 ns      2007040
task_benchmark<fi::fptr_task_1, true>             434 ns          460 ns      2036364
```

These results might be a bit unexpected. Our custom vtable implementations seem to be the fastest. Then comes our virtual task implementation and the last one is our `std::move_only_function` based implementation. It's worth noting here, that Microsoft's implementation of STL which I'm using performs some sort of small buffer optimization. The shallow size of `std::move_only_function<void()>` is 64 bytes.

Complete implementation can be found [here](https://github.com/RedSkittleFox/thread_pool/blob/main/benchmark/task_benchmark.cc).

## Implementing deque buffer
Firstly we will implement the simplest form of buffer - `std::deque` based queue that uses locks to 

![_config.yml]({{ site.baseurl }}/images/config.png)

The easiest way to make your first post is to edit this one. Go into /_posts/ and update the Hello World markdown file. For more instructions head over to the [Jekyll Now repository](https://github.com/barryclark/jekyll-now) on GitHub.