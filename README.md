# Condition
Your task is to find as many mistakes and drawbacks in this code (according to the theme material) as you can. Annotate these mistakes with comments.

# Restrictions
- MyVector stores a collection of objects with their names. For each object `T`, `MyVector` stores T's name as `std::string`.  
- Several objects can have similar name.
- `operator[](const std::string& name)` should return the first object with the given name.

# Need to implement
Once you have found all the mistakes, rewrite the code so it would not change its original purpose and it would contain no mistakes. Try to make the code more efficient without premature optimization.  
You can change MyVector interface completely, but there are several rules:
1. You should correctly and fully implement **copy-on-write** idiom.
2. `std::pair<const T&, const std::string&> operator[](int index) const` must take constant time at worst.
3. `const T& operator[](const std::string& name) const` should be present.
4. `both operator[]` should have non-const version.
5. Your implementation should provide all the member types of `std::vector`.
6. Your implementation should provide the following functions:
    - `begin(), cbegin(), end(), cend()`
    - `empty(), size()`
    - `reserve(), clear()`

# Code for review and refinement
~~~C++
#ifndef CODEREVIEWTASK_MYVECTOR_HPP
#define CODEREVIEWTASK_MYVECTOR_HPP

#include <vector>
#include <string>
#include <algorithm>
#include <stdexcept>

template <typename T>
class MyVector : public std::vector<T>
{
public:
    MyVector()
    {
        m_ref_ptr = new size_t(1);
        m_names = new std::vector<std::string>();
    }

    MyVector(const MyVector& other)
        : std::vector<T>(other),
          m_ref_ptr(other.m_ref_ptr),
          m_names(other.m_names)
    {
        (*m_ref_ptr)++;
    }

    ~MyVector()
    {
        if (--*m_ref_ptr == 0)
        {
            delete m_ref_ptr;
            delete m_names;
        }
    }

    void push_back(const T& obj, const std::string& name)
    {
        copy_names();

        std::vector<T>::push_back(obj);
        m_names->push_back(name);
    }

    std::pair<const T&, const std::string&> operator[](int index) const
    {
        if (index >= std::vector<T>::size())
        {
            throw new std::out_of_range("Index is out of range");
        }

        return std::pair<const T&, const std::string&>(std::vector<T>::operator[](index), (*m_names)[index]);
    }

    const T& operator[](const std::string& name) const
    {
        std::vector<std::string>::const_iterator iter = std::find(m_names->begin(), m_names->end(), name);
        if (iter == m_names->end())
        {
            throw new std::invalid_argument(name + " is not found in the MyVector");
        }

        return std::vector<T>::operator[](iter - m_names->begin());
    }

private:
    void copy_names()
    {
        if (*m_ref_ptr == 1)
        {
            return;
        }

        size_t* temp_ref_ptr = new size_t(1);
        std::vector<std::string>* temp_names = new std::vector<std::string>(*m_names);

        (*m_ref_ptr)--;
        m_ref_ptr = temp_ref_ptr;
        m_names = temp_names;
    }

private:
    // Use copy-on-write idiom for efficiency (not a premature optimization)
    std::vector<std::string>* m_names;
    size_t* m_ref_ptr;
};


#endif //CODEREVIEWTASK_MYVECTOR_HPP
~~~

# Delivery method
- Base code with comments describing the errors found
- Modified project

# Needs fixes
- ***Smart pointers are not used***

Removed a structure containing a data vector, a vector of names, as well as a bare pointer. In the new implementation, the data and it names are paired and added to the vector. 

~~~C++
using vector_data = std::vector<std::pair<T, std::string>>;	//save pair of T data and string name
~~~

The result is wrapped in a shared_ptr smart pointer
~~~C++
std::shared_ptr<vector_data> m_data{};
~~~
___

- ***Methods like empy, size must be constant***

Unfortunately, there was indeed such an omission in the initial implementation. All methods that do not change data are made constant

~~~C++
//return if data vector is empty
template <typename T>
bool MyVector<T>::empty() const {
	return m_data->empty();
}
~~~
~~~C++
//return size of data vector
template <typename T>
size_t MyVector<T>::size() const {
	return m_data->size();
}
~~~
___
- ***Assignment operator will not work correctly***

Copy assignment operator and move assignment operator are prohibited in solution

~~~C++
MyVector& operator =(const MyVector& other) = delete;	//denied operator =
MyVector& operator =(MyVector&& other) = delete;		//denied operator =
~~~
____

- No strong exception safety guarantee

Unfortunately, given the tight deadlines, there was not enough time to conduct an in-depth analysis of each method. Part of the methods that change the data structure are indicated by a guarantee of no exception

~~~C++
//add new element in vector
void push_back(const T& obj, const std::string& name) noexcept;	

//Return iterator to fisrt element of data vector	
typename std::vector<std::pair<T, std::string>>::iterator begin() noexcept;	

//Return iterator to the end of data vector	
typename std::vector<std::pair<T, std::string>>::iterator end() noexcept;		
~~~
____
- All non-const methods must create a copy

Probably some of the methods of the original solution did not satisfy this principle. Considering the change in the structure of the data stored in the class, the obligatory data copying was controlled

~~~C++
void copy_datas();	//copy data vector, in case of operation write
~~~
____
- Proxy class is not needed, you can immediately make a copy in non-const methods

The question arises, witch role playing those methods without using a class proxy. In its current form, the const method never seems to run. However, the proxy class is excluded from the final solution

~~~C++
//const overload [size_t]
template <typename T>
const std::pair<const T&, const std::string&> MyVector<T>::operator[](size_t index) const
{
	if (index >= this->m_data->size() || index < 0)
	{
		throw std::out_of_range("Index is out of range");
	}
	return this->m_data->operator[](index);
}
~~~

~~~C++
//overload [size_t]
template <typename T>
std::pair<T, std::string>& MyVector<T>::operator[](size_t index)
{
	if (index >= this->m_data->size() || index < 0)
	{
		throw std::out_of_range("Index is out of range");
	}
	copy_datas();	//becouse changed data
	return this->m_data->operator[](index);
}
~~~
