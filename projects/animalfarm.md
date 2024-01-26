---
layout: project
type: project
image: img/animalfarm.jpg
title: "Animal Farm"
date: 2023
published: true
labels:
  - CLion
  - C++
summary: "This is an object-oriented programming project built in CLion using C++. The idea is to build a farm with animals that have age, color, types, species, genders, ID numbers, and other traits"
---

In the Spring 2023 semester, I enrolled in EE 205 with Professor Mark Nelson. Throughout the semester, we worked on a C++ Project on CLion in which we created a farm that sheltered animals. Common concepts that I gained significant experience in were Classes, constructors and destructors, singly linked-lists, testing using Boost.Test, virtual functions, method overriding, public vs. private vs. protected members, and inheritance. Unfortunately, the Github repository is now gone, but here is some sample code for a general idea:

### Animal Class

```cpp
#include <iostream>
#include <string>

class Animal {
public:
    Animal(int id, int age, const std::string& color, const std::string& type, const std::string& species, const std::string& gender)
        : id(id), age(age), color(color), type(type), species(species), gender(gender) {}

    void display() const {
        std::cout << "ID: " << id << ", Age: " << age << ", Color: " << color
                  << ", Type: " << type << ", Species: " << species << ", Gender: " << gender << std::endl;
    }

private:
    int id;
    int age;
    std::string color;
    std::string type;
    std::string species;
    std::string gender;
};
```

### AnimalNode and AnimalList Classes

```cpp
class AnimalNode {
public:
    AnimalNode(Animal animal) : animal(animal), next(nullptr) {}

private:
    Animal animal;
    AnimalNode* next;

    friend class AnimalList; // AnimalList class will have access to private members of AnimalNode
};

class AnimalList {
public:
    AnimalList() : head(nullptr) {}

    void addAnimal(const Animal& animal) {
        AnimalNode* newNode = new AnimalNode(animal);
        newNode->next = head;
        head = newNode;
    }

    void displayAll() const {
        AnimalNode* current = head;
        while (current) {
            current->animal.display();
            current = current->next;
        }
    }

    ~AnimalList() {
        // Implement a destructor to free memory used by linked list nodes
        AnimalNode* current = head;
        while (current) {
            AnimalNode* next = current->next;
            delete current;
            current = next;
        }
        head = nullptr;
    }

private:
    AnimalNode* head;
};
```

### Main Program

```cpp
int main() {
    AnimalList farm;

    farm.addAnimal(Animal(1, 5, "Brown", "Dog", "Canine", "Male"));
    farm.addAnimal(Animal(2, 2, "Black", "Cat", "Feline", "Female"));
    farm.addAnimal(Animal(3, 1, "White", "Chicken", "Avian", "Female"));

    std::cout << "All Animals on the Farm:" << std::endl;
    farm.displayAll();

    return 0;
}
```
