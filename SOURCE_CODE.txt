"""
Student Course Management System
A secure, object-oriented console application for recording semester courses.

Python 3.9+
Storage: JSON (human-readable; no unsafe pickle deserialization)
"""

from __future__ import annotations

import json
import re
import tempfile
from dataclasses import asdict, dataclass
from pathlib import Path
from typing import List, Optional


DATA_FILE = Path(__file__).with_name("courses.json")
MAX_DATA_FILE_BYTES = 1_000_000
MAX_COURSES = 500
COURSE_CODE_PATTERN = re.compile(r"^[A-Z]{2,5}\s?\d{2,4}[A-Z]?$")


@dataclass
class Course:
    """Represents one course registered by a student."""

    code: str
    title: str
    units: int

    def __post_init__(self) -> None:
        self.code = self.code.strip().upper()
        self.title = " ".join(self.title.strip().split())
        if not COURSE_CODE_PATTERN.fullmatch(self.code):
            raise ValueError("Course code must look like CSC201 or CSC 201.")
        if not self.title or len(self.title) > 100:
            raise ValueError("Course title must contain 1–100 characters.")
        if not isinstance(self.units, int) or isinstance(self.units, bool):
            raise ValueError("Credit units must be a whole number.")
        if not 1 <= self.units <= 10:
            raise ValueError("Credit units must be between 1 and 10.")


class CourseManager:
    """Stores and manages the student's course records."""

    def __init__(self, data_file: Path = DATA_FILE) -> None:
        self._courses: List[Course] = []
        self.data_file = Path(data_file)

    @property
    def courses(self) -> List[Course]:
        # Return a copy so callers cannot alter the internal collection.
        return list(self._courses)

    def add_course(self, course: Course) -> None:
        if len(self._courses) >= MAX_COURSES:
            raise ValueError(f"You can record a maximum of {MAX_COURSES} courses.")
        if self.search_course(course.code):
            raise ValueError(f"Course {course.code} already exists.")
        self._courses.append(course)

    def search_course(self, code: str) -> Optional[Course]:
        normalized = code.strip().upper()
        for course in self._courses:
            if course.code == normalized:
                return course
        return None

    def total_units(self) -> int:
        """Calculate total units recursively, as required by the project."""
        def recursive_sum(index: int) -> int:
            if index >= len(self._courses):  # Base case
                return 0
            return self._courses[index].units + recursive_sum(index + 1)

        return recursive_sum(0)

    def save_to_file(self) -> None:
        """Write validated records using an atomic replacement."""
        payload = {
            "format_version": 1,
            "courses": [asdict(course) for course in self._courses],
        }
        self.data_file.parent.mkdir(parents=True, exist_ok=True)
        temp_name = None
        try:
            with tempfile.NamedTemporaryFile(
                mode="w", encoding="utf-8", dir=self.data_file.parent,
                delete=False, suffix=".tmp"
            ) as temp_file:
                temp_name = temp_file.name
                json.dump(payload, temp_file, indent=4, ensure_ascii=False)
                temp_file.write("\n")
                temp_file.flush()
                os_fsync = getattr(temp_file, "fileno", None)
                if os_fsync:
                    import os
                    os.fsync(temp_file.fileno())
            Path(temp_name).replace(self.data_file)
        except OSError:
            if temp_name and Path(temp_name).exists():
                Path(temp_name).unlink(missing_ok=True)
            raise

    def load_from_file(self) -> int:
        """Load and validate all records before replacing current data."""
        if not self.data_file.exists():
            raise FileNotFoundError(f"No saved file found at: {self.data_file}")
        if self.data_file.stat().st_size > MAX_DATA_FILE_BYTES:
            raise ValueError("The data file is too large to load safely.")

        try:
            with self.data_file.open("r", encoding="utf-8") as file:
                payload = json.load(file)
        except json.JSONDecodeError as exc:
            raise ValueError("The saved file is not valid JSON.") from exc

        if not isinstance(payload, dict) or payload.get("format_version") != 1:
            raise ValueError("Unsupported or invalid data-file format.")
        records = payload.get("courses")
        if not isinstance(records, list) or len(records) > MAX_COURSES:
            raise ValueError("Course list is missing, invalid, or too large.")

        validated: List[Course] = []
        seen_codes = set()
        for record in records:
            if not isinstance(record, dict) or set(record) != {"code", "title", "units"}:
                raise ValueError("A course record has missing or unexpected fields.")
            course = Course(
                code=str(record["code"]),
                title=str(record["title"]),
                units=record["units"],
            )
            if course.code in seen_codes:
                raise ValueError(f"Duplicate course code in file: {course.code}")
            seen_codes.add(course.code)
            validated.append(course)

        # Replace current records only after the entire file passes validation.
        self._courses = validated
        return len(validated)


class StudentCourseApp:
    """Text-based user interface."""

    def __init__(self) -> None:
        self.manager = CourseManager()

    @staticmethod
    def show_menu() -> None:
        print("\n" + "=" * 56)
        print("          STUDENT COURSE MANAGEMENT SYSTEM")
        print("=" * 56)
        print("  1. Add Course")
        print("  2. View All Courses")
        print("  3. Search Course by Code")
        print("  4. Compute Total Units")
        print("  5. Save to File")
        print("  6. Load from File")
        print("  7. Exit Program")
        print("-" * 56)

    @staticmethod
    def read_non_empty(prompt: str, max_length: int) -> str:
        value = input(prompt).strip()
        if not value:
            raise ValueError("This field cannot be empty.")
        if len(value) > max_length:
            raise ValueError(f"Input must not exceed {max_length} characters.")
        return value

    def add_course(self) -> None:
        code = self.read_non_empty("Course code (e.g. CSC201): ", 10)
        title = self.read_non_empty("Course title: ", 100)
        units_text = self.read_non_empty("Credit units (1-10): ", 2)
        if not units_text.isdigit():
            raise ValueError("Credit units must be a whole number.")
        course = Course(code=code, title=title, units=int(units_text))
        self.manager.add_course(course)
        print(f"\n[SUCCESS] {course.code} - {course.title} added.")

    def view_courses(self) -> None:
        courses = self.manager.courses
        if not courses:
            print("\nNo courses have been recorded yet.")
            return
        print("\n" + "-" * 72)
        print(f"{'NO.':<5}{'COURSE CODE':<16}{'COURSE TITLE':<39}{'UNITS':>8}")
        print("-" * 72)
        for number, course in enumerate(courses, start=1):
            print(f"{number:<5}{course.code:<16}{course.title[:37]:<39}{course.units:>8}")
        print("-" * 72)
        print(f"Courses recorded: {len(courses)}")

    def search_course(self) -> None:
        code = self.read_non_empty("Enter course code to search: ", 10).upper()
        course = self.manager.search_course(code)
        if course:
            print("\n[FOUND]")
            print(f"Course code : {course.code}")
            print(f"Course title: {course.title}")
            print(f"Credit units: {course.units}")
        else:
            print(f"\n[NOT FOUND] No course with code {code} exists.")

    def compute_total(self) -> None:
        print(f"\nTotal registered courses: {len(self.manager.courses)}")
        print(f"Total credit units       : {self.manager.total_units()}")

    def save_courses(self) -> None:
        self.manager.save_to_file()
        print(f"\n[SUCCESS] {len(self.manager.courses)} course(s) saved to {self.manager.data_file.name}.")

    def load_courses(self) -> None:
        count = self.manager.load_from_file()
        print(f"\n[SUCCESS] {count} course(s) loaded from {self.manager.data_file.name}.")

    def run(self) -> None:
        print("\nWelcome. Manage your semester courses in one place.")
        while True:
            self.show_menu()
            choice = input("Select an option (1-7): ").strip()
            try:
                if choice == "1":
                    self.add_course()
                elif choice == "2":
                    self.view_courses()
                elif choice == "3":
                    self.search_course()
                elif choice == "4":
                    self.compute_total()
                elif choice == "5":
                    self.save_courses()
                elif choice == "6":
                    self.load_courses()
                elif choice == "7":
                    print("\nThank you for using the Student Course Management System.")
                    print("Goodbye!")
                    break
                else:
                    print("\n[INVALID OPTION] Enter a number from 1 to 7.")
            except (ValueError, FileNotFoundError, PermissionError, OSError) as error:
                print(f"\n[ERROR] {error}")
            except KeyboardInterrupt:
                print("\n\nProgram interrupted safely. Goodbye!")
                break


if __name__ == "__main__":
    StudentCourseApp().run()
