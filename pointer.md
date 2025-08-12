# Pointer and Reference

## Raw Pointer (Con trỏ)
- Pointer là một biến lưu trữ địa chỉ của một biến khác trong bộ nhớ.
- Con trỏ cho phép truy cập và thao tác trực tiếp với dữ liệu thông qua địa chỉ.
- Cú pháp:
```cpp
type* pointerName; // Khai báo con trỏ
```
```cpp
#include <iostream>
int main() {
    int x = 10;       // Biến thông thường
    int* ptr = &x;    // Con trỏ lưu địa chỉ của x
    std::cout << "Value of x: " << x << std::endl;
    std::cout << "Address of x: " << &x << std::endl;
    std::cout << "Value stored in ptr: " << ptr << std::endl;
    std::cout << "Value pointed by ptr: " << *ptr << std::endl;
    return 0;
}
/*
&x: Lấy địa chỉ của biến x.
ptr: Lưu địa chỉ của x.
*ptr: Truy cập giá trị tại địa chỉ mà con trỏ ptr trỏ tới.
*/
```

- Các thao tác với Pointer:

|Thao tác|Note|
|---|---|
|Gán giá trị| int a = 5; int* p = &a; // Con trỏ p trỏ tới biến a|
|Truy cập giá trị|std::cout << *p; // Giá trị của biến a thông qua con trỏ p|
|Thay đổi giá trị thông qua con trỏ|*p = 10; // Thay đổi giá trị của a thông qua con trỏ p|
|Con trỏ NULL|int* p = nullptr; // Con trỏ không trỏ tới bất kỳ địa chỉ nào|

## Reference (Tham chiếu)
- Reference là một alias (bí danh) cho một biến. Nó không lưu địa chỉ như con trỏ, mà trực tiếp tham chiếu đến biến gốc.
- Reference phải được khởi tạo khi khai báo và không thể thay đổi để tham chiếu đến biến khác.
- Cú pháp:
```cpp
type& referenceName = variableName; // Khai báo tham chiếu
```
```cpp
#include <iostream>
int main() {
    int x = 10;
    int& ref = x; // Tham chiếu ref tới biến x
    std::cout << "Value of x: " << x << std::endl;
    std::cout << "Value of ref: " << ref << std::endl;
    ref = 20; // Thay đổi giá trị của x thông qua ref
    std::cout << "Value of x after modification: " << x << std::endl;
    return 0;
}
/*
int& ref = x;: ref là tham chiếu đến biến x.
Thay đổi giá trị của ref cũng thay đổi giá trị của x.
*/
```

## So sánh Pointer và Reference
|Tiêu chí|Pointer|Reference|
|-|-|-|
|Lưu trữ|Lưu địa chỉ của biến|Là bí danh trực tiếp của biến|
|Khởi tạo|Có thể khai báo mà không cần khởi tạo|Phải khởi tạo khi khai báo|
|Thay dổi mục tiêu|Có thể trỏ tới biến khác|Không thể thay đổi biến được tham chiếu|
|Truy cập giá trị|Sử dụng * để truy cập giá trị|Truy cập trực tiếp như biến thông thường|


```cpp
#include <iostream>
void modifyValue(int* p) {
    *p = 20; // Thay đổi giá trị thông qua con trỏ
}
int main() {
    int x = 10;
    modifyValue(&x); // Truyền địa chỉ của x
    std::cout << "Value of x: " << x << std::endl;
    return 0;
}
```

```cpp
#include <iostream>
void modifyValue(int& ref) {
    ref = 20; // Thay đổi giá trị thông qua tham chiếu
}
int main() {
    int x = 10;
    modifyValue(x); // Truyền biến x
    std::cout << "Value of x: " << x << std::endl;
    return 0;
}
```

# Smart pointer
- Smart pointers là các lớp trong C++ được thiết kế để quản lý bộ nhớ tự động, giúp tránh các lỗi như memory leaks

- std::shared_ptr: Dùng khi cần chia sẻ quyền sở hữu đối tượng giữa nhiều con trỏ.
- std::unique_ptr: Dùng khi cần đảm bảo quyền sở hữu duy nhất đối với đối tượng.
- std::weak_ptr: Dùng để tránh vòng tham chiếu khi làm việc với std::shared_ptr.

```cpp
/*
Dùng shared_ptr để chia sẻ ownership của Department và Employee.

Dùng weak_ptr để tránh vòng lặp ownership (giữa Employee và Department).

Dùng unique_ptr để quản lý Company, vì chỉ có duy nhất một nơi sở hữu.
*/
#include <iostream>
#include <memory>
#include <vector>
#include <string>

class Department; // forward declaration

class Employee {
    std::string name;
    std::weak_ptr<Department> department;  // dùng weak_ptr để tránh vòng

public:
    Employee(const std::string& name) : name(name) {}

    void setDepartment(std::shared_ptr<Department> dept) {
        department = dept; // không tăng ref count
    }

    void show() const {
        std::cout << "👤 Employee: " << name;
        if (auto dept = department.lock()) {
            std::cout << " (Dept: " << dept->getName() << ")";
        } else {
            std::cout << " (No Department)";
        }
        std::cout << std::endl;
    }
};

class Department : public std::enable_shared_from_this<Department> {
    std::string name;
    std::vector<std::shared_ptr<Employee>> employees;

public:
    Department(const std::string& name) : name(name) {}

    const std::string& getName() const { return name; }

    void addEmployee(std::shared_ptr<Employee> emp) {
        emp->setDepartment(shared_from_this());  // weak_ptr gán ngược
        employees.push_back(emp);
    }

    void show() const {
        std::cout << "🏢 Department: " << name << std::endl;
        for (const auto& emp : employees) {
            emp->show();
        }
    }
};

class Company {
    std::string name;
    std::vector<std::shared_ptr<Department>> departments;

public:
    Company(const std::string& name) : name(name) {}

    void addDepartment(std::shared_ptr<Department> dept) {
        departments.push_back(dept);
    }

    void show() const {
        std::cout << "\n🏬 Company: " << name << "\n";
        for (const auto& dept : departments) {
            dept->show();
        }
    }
};

int main() {
    // Company được quản lý bằng unique_ptr → chỉ có 1 chủ sở hữu
    std::unique_ptr<Company> company = std::make_unique<Company>("ABC");

    // Dùng shared_ptr cho các phòng ban (có thể chia sẻ nhiều nơi)
    auto devDept = std::make_shared<Department>("Development");
    auto hrDept  = std::make_shared<Department>("HR");

    // Nhân viên cũng dùng shared_ptr
    auto alice = std::make_shared<Employee>("Alice");
    auto bob   = std::make_shared<Employee>("Bob");
    auto john  = std::make_shared<Employee>("John");

    // Gán nhân viên vào phòng ban
    devDept->addEmployee(alice);
    devDept->addEmployee(bob);
    hrDept->addEmployee(john);

    // Thêm phòng ban vào công ty
    company->addDepartment(devDept);
    company->addDepartment(hrDept);

    // Hiển thị toàn bộ hệ thống
    company->show();

    return 0; // Khi thoát scope: unique_ptr → shared_ptr → weak_ptr đều tự hủy
}

```