# Requirements – Testownik

**Project:** Testownik
**Author:** Antoni Czaplicki

---

## 1. Purpose and Scope

Testownik is a quiz-based learning and knowledge verification platform created for WUST students.
We aim to provide an engaging and effective tool for students to prepare for exams, practice problem-solving, and receive personalized feedback.

---

## 2. Actors

| Actor                  | Description                                                                                               |
|------------------------|-----------------------------------------------------------------------------------------------------------|
| **Guest**              | Unauthenticated user with a server-side anonymous session (`AccountType.GUEST`)                           |
| **Authenticated user** | User with an email or USOS account (`AccountType.EMAIL` / `AccountType.STUDENT` / `AccountType.LECTURER`) |
| **AI system**          | External or internal ML service providing hints and generating questions                                  |
| **Administrator**      | Staff user managing the platform via Django Admin                                                         |
