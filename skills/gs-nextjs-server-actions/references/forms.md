# Forms Reference

Complete guide to handling forms with ZSA server actions.

## Basic Form with FormData

```typescript
// actions/contact.ts
"use server";

import { createServerAction } from "zsa";
import z from "zod";

export const submitContactAction = createServerAction()
  .input(
    z.object({
      name: z.string().min(2, "Name must be at least 2 characters"),
      email: z.string().email("Invalid email address"),
      message: z.string().min(10, "Message must be at least 10 characters"),
    }),
    { type: "formData" } // Enable FormData parsing
  )
  .handler(async ({ input }) => {
    await sendEmail({
      to: "support@example.com",
      subject: `Contact from ${input.name}`,
      body: input.message,
      replyTo: input.email,
    });
    return { success: true };
  });
```

```typescript
// components/contact-form.tsx
"use client";

import { useServerAction } from "zsa-react";
import { submitContactAction } from "@/actions/contact";

export function ContactForm() {
  const { isPending, executeFormAction, isSuccess, isError, error, reset } =
    useServerAction(submitContactAction);

  if (isSuccess) {
    return (
      <div>
        <p>Thank you for your message!</p>
        <button onClick={reset}>Send another</button>
      </div>
    );
  }

  return (
    <form action={executeFormAction}>
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" name="name" required />
        {isError && error.fieldErrors?.name && (
          <span className="error">{error.fieldErrors.name[0]}</span>
        )}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" required />
        {isError && error.fieldErrors?.email && (
          <span className="error">{error.fieldErrors.email[0]}</span>
        )}
      </div>

      <div>
        <label htmlFor="message">Message</label>
        <textarea id="message" name="message" required />
        {isError && error.fieldErrors?.message && (
          <span className="error">{error.fieldErrors.message[0]}</span>
        )}
      </div>

      <button type="submit" disabled={isPending}>
        {isPending ? "Sending..." : "Send Message"}
      </button>

      {isError && !error.fieldErrors && (
        <p className="error">{error.message}</p>
      )}
    </form>
  );
}
```

## Form with React Hook Form

```typescript
// actions/register.ts
"use server";

import { createServerAction } from "zsa";
import z from "zod";

export const registerSchema = z
  .object({
    email: z.string().email(),
    password: z.string().min(8),
    confirmPassword: z.string(),
    terms: z.boolean().refine((v) => v === true, "Must accept terms"),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords don't match",
    path: ["confirmPassword"],
  });

export type RegisterInput = z.infer<typeof registerSchema>;

export const registerAction = createServerAction()
  .input(registerSchema)
  .handler(async ({ input }) => {
    const user = await createUser({
      email: input.email,
      password: await hashPassword(input.password),
    });
    return { userId: user.id };
  });
```

```typescript
// components/register-form.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { useServerAction } from "zsa-react";
import { registerAction, registerSchema, type RegisterInput } from "@/actions/register";

export function RegisterForm() {
  const { execute, isPending, isError, error } = useServerAction(registerAction);

  const form = useForm<RegisterInput>({
    resolver: zodResolver(registerSchema),
    defaultValues: {
      email: "",
      password: "",
      confirmPassword: "",
      terms: false,
    },
  });

  const onSubmit = async (data: RegisterInput) => {
    const [result, err] = await execute(data);

    if (err) {
      // Handle server-side errors
      if (err.code === "INPUT_PARSE_ERROR" && err.fieldErrors) {
        // Set field errors from server
        Object.entries(err.fieldErrors).forEach(([field, errors]) => {
          if (errors?.[0]) {
            form.setError(field as keyof RegisterInput, {
              message: errors[0],
            });
          }
        });
      } else {
        // General error
        form.setError("root", { message: err.message });
      }
      return;
    }

    // Success - redirect or show message
    router.push("/dashboard");
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <div>
        <input
          {...form.register("email")}
          type="email"
          placeholder="Email"
          disabled={isPending}
        />
        {form.formState.errors.email && (
          <span className="error">{form.formState.errors.email.message}</span>
        )}
      </div>

      <div>
        <input
          {...form.register("password")}
          type="password"
          placeholder="Password"
          disabled={isPending}
        />
        {form.formState.errors.password && (
          <span className="error">{form.formState.errors.password.message}</span>
        )}
      </div>

      <div>
        <input
          {...form.register("confirmPassword")}
          type="password"
          placeholder="Confirm Password"
          disabled={isPending}
        />
        {form.formState.errors.confirmPassword && (
          <span className="error">
            {form.formState.errors.confirmPassword.message}
          </span>
        )}
      </div>

      <div>
        <label>
          <input {...form.register("terms")} type="checkbox" disabled={isPending} />
          I accept the terms and conditions
        </label>
        {form.formState.errors.terms && (
          <span className="error">{form.formState.errors.terms.message}</span>
        )}
      </div>

      {form.formState.errors.root && (
        <p className="error">{form.formState.errors.root.message}</p>
      )}

      <button type="submit" disabled={isPending}>
        {isPending ? "Creating account..." : "Register"}
      </button>
    </form>
  );
}
```

## File Upload

```typescript
// actions/upload.ts
"use server";

import { createServerAction } from "zsa";
import z from "zod";
import { writeFile } from "fs/promises";
import { join } from "path";

const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
const ACCEPTED_TYPES = ["image/jpeg", "image/png", "image/webp"];

export const uploadImageAction = createServerAction()
  .input(
    z.object({
      file: z
        .instanceof(File)
        .refine((f) => f.size <= MAX_FILE_SIZE, "Max file size is 5MB")
        .refine(
          (f) => ACCEPTED_TYPES.includes(f.type),
          "Only .jpg, .png, .webp files are accepted"
        ),
      caption: z.string().optional(),
    }),
    { type: "formData" }
  )
  .handler(async ({ input }) => {
    const { file, caption } = input;

    // Generate unique filename
    const ext = file.name.split(".").pop();
    const filename = `${Date.now()}-${Math.random().toString(36).slice(2)}.${ext}`;
    const filepath = join(process.cwd(), "public/uploads", filename);

    // Save file
    const buffer = Buffer.from(await file.arrayBuffer());
    await writeFile(filepath, buffer);

    // Save to database
    const image = await db.image.create({
      data: {
        url: `/uploads/${filename}`,
        caption,
        originalName: file.name,
        size: file.size,
        mimeType: file.type,
      },
    });

    return image;
  });
```

```typescript
// components/image-upload.tsx
"use client";

import { useServerAction } from "zsa-react";
import { uploadImageAction } from "@/actions/upload";
import { useState, useRef } from "react";

export function ImageUpload() {
  const fileInputRef = useRef<HTMLInputElement>(null);
  const [preview, setPreview] = useState<string | null>(null);

  const { execute, isPending, isError, error, data, reset } =
    useServerAction(uploadImageAction);

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) {
      setPreview(URL.createObjectURL(file));
    }
  };

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const file = formData.get("file") as File;
    const caption = formData.get("caption") as string;

    const [result, err] = await execute({ file, caption });

    if (!err) {
      // Clear form on success
      e.currentTarget.reset();
      setPreview(null);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          ref={fileInputRef}
          type="file"
          name="file"
          accept="image/jpeg,image/png,image/webp"
          onChange={handleFileChange}
          disabled={isPending}
        />
        {isError && error.fieldErrors?.file && (
          <span className="error">{error.fieldErrors.file[0]}</span>
        )}
      </div>

      {preview && (
        <img src={preview} alt="Preview" style={{ maxWidth: 200 }} />
      )}

      <div>
        <input
          type="text"
          name="caption"
          placeholder="Caption (optional)"
          disabled={isPending}
        />
      </div>

      <button type="submit" disabled={isPending || !preview}>
        {isPending ? "Uploading..." : "Upload"}
      </button>

      {data && (
        <div>
          <p>Uploaded successfully!</p>
          <img src={data.url} alt={data.caption} style={{ maxWidth: 200 }} />
        </div>
      )}
    </form>
  );
}
```

## Multiple File Upload

```typescript
// actions/upload-multiple.ts
"use server";

import { createServerAction } from "zsa";
import z from "zod";

export const uploadMultipleAction = createServerAction()
  .input(
    z.object({
      files: z
        .array(z.instanceof(File))
        .min(1, "Select at least one file")
        .max(5, "Maximum 5 files allowed"),
      albumId: z.string(),
    }),
    { type: "formData" }
  )
  .handler(async ({ input }) => {
    const uploadedFiles = await Promise.all(
      input.files.map(async (file) => {
        const url = await uploadToStorage(file);
        return db.file.create({
          data: {
            url,
            name: file.name,
            albumId: input.albumId,
          },
        });
      })
    );
    return uploadedFiles;
  });
```

```typescript
// components/multi-upload.tsx
"use client";

import { useServerAction } from "zsa-react";
import { uploadMultipleAction } from "@/actions/upload-multiple";

export function MultiUpload({ albumId }: { albumId: string }) {
  const { execute, isPending, isError, error, data } =
    useServerAction(uploadMultipleAction);

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const files = formData.getAll("files") as File[];

    await execute({ files, albumId });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="file"
        name="files"
        multiple
        accept="image/*"
        disabled={isPending}
      />
      <button type="submit" disabled={isPending}>
        {isPending ? "Uploading..." : "Upload Files"}
      </button>
      {isError && <p className="error">{error.message}</p>}
      {data && <p>Uploaded {data.length} files!</p>}
    </form>
  );
}
```

## Dynamic Form Fields

```typescript
// actions/survey.ts
"use server";

import { createServerAction } from "zsa";
import z from "zod";

export const submitSurveyAction = createServerAction()
  .input(
    z.object({
      answers: z.array(
        z.object({
          questionId: z.string(),
          value: z.string(),
        })
      ),
    }),
    { type: "formData" }
  )
  .handler(async ({ input }) => {
    return db.surveyResponse.create({
      data: { answers: input.answers },
    });
  });
```

```typescript
// components/survey-form.tsx
"use client";

import { useServerAction } from "zsa-react";
import { submitSurveyAction } from "@/actions/survey";
import { useFieldArray, useForm } from "react-hook-form";

interface Question {
  id: string;
  text: string;
}

export function SurveyForm({ questions }: { questions: Question[] }) {
  const { execute, isPending } = useServerAction(submitSurveyAction);
  
  const form = useForm({
    defaultValues: {
      answers: questions.map((q) => ({ questionId: q.id, value: "" })),
    },
  });

  const { fields } = useFieldArray({
    control: form.control,
    name: "answers",
  });

  const onSubmit = async (data: { answers: { questionId: string; value: string }[] }) => {
    await execute(data);
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {fields.map((field, index) => (
        <div key={field.id}>
          <label>{questions[index].text}</label>
          <input {...form.register(`answers.${index}.value`)} />
          <input
            type="hidden"
            {...form.register(`answers.${index}.questionId`)}
          />
        </div>
      ))}
      <button type="submit" disabled={isPending}>
        {isPending ? "Submitting..." : "Submit"}
      </button>
    </form>
  );
}
```

## Form with Nested Objects

```typescript
// actions/profile.ts
"use server";

import { createServerAction } from "zsa";
import z from "zod";

export const updateProfileAction = createServerAction()
  .input(
    z.object({
      name: z.string(),
      bio: z.string().optional(),
      address: z.object({
        street: z.string(),
        city: z.string(),
        country: z.string(),
        zip: z.string(),
      }),
      social: z.object({
        twitter: z.string().url().optional().or(z.literal("")),
        github: z.string().url().optional().or(z.literal("")),
        linkedin: z.string().url().optional().or(z.literal("")),
      }),
    }),
    { type: "formData" }
  )
  .handler(async ({ input }) => {
    return db.user.update({
      where: { id: currentUser.id },
      data: input,
    });
  });
```

```typescript
// components/profile-form.tsx
"use client";

import { useServerAction } from "zsa-react";
import { updateProfileAction } from "@/actions/profile";

export function ProfileForm({ user }: { user: User }) {
  const { executeFormAction, isPending, isError, error, isSuccess } =
    useServerAction(updateProfileAction);

  return (
    <form action={executeFormAction}>
      <fieldset>
        <legend>Basic Info</legend>
        <input name="name" defaultValue={user.name} />
        <textarea name="bio" defaultValue={user.bio} />
      </fieldset>

      <fieldset>
        <legend>Address</legend>
        <input name="address.street" defaultValue={user.address?.street} />
        <input name="address.city" defaultValue={user.address?.city} />
        <input name="address.country" defaultValue={user.address?.country} />
        <input name="address.zip" defaultValue={user.address?.zip} />
      </fieldset>

      <fieldset>
        <legend>Social Links</legend>
        <input name="social.twitter" defaultValue={user.social?.twitter} />
        <input name="social.github" defaultValue={user.social?.github} />
        <input name="social.linkedin" defaultValue={user.social?.linkedin} />
      </fieldset>

      <button type="submit" disabled={isPending}>
        {isPending ? "Saving..." : "Save Profile"}
      </button>

      {isSuccess && <p className="success">Profile updated!</p>}
      {isError && <p className="error">{error.message}</p>}
    </form>
  );
}
```

## Form Validation Patterns

### Client + Server Validation

```typescript
// lib/validations/user.ts
import z from "zod";

// Shared schema for client and server
export const createUserSchema = z.object({
  email: z.string().email("Please enter a valid email"),
  password: z
    .string()
    .min(8, "Password must be at least 8 characters")
    .regex(/[A-Z]/, "Password must contain an uppercase letter")
    .regex(/[0-9]/, "Password must contain a number"),
  name: z.string().min(2, "Name must be at least 2 characters"),
});

export type CreateUserInput = z.infer<typeof createUserSchema>;
```

```typescript
// actions/user.ts
"use server";

import { createServerAction } from "zsa";
import { createUserSchema } from "@/lib/validations/user";

export const createUserAction = createServerAction()
  .input(createUserSchema)
  .handler(async ({ input }) => {
    // Check if email exists (server-only validation)
    const existing = await db.user.findUnique({
      where: { email: input.email },
    });
    if (existing) {
      throw new Error("Email already in use");
    }
    return db.user.create({ data: input });
  });
```

```typescript
// components/create-user-form.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { createUserSchema, type CreateUserInput } from "@/lib/validations/user";
import { useServerAction } from "zsa-react";
import { createUserAction } from "@/actions/user";

export function CreateUserForm() {
  const { execute, isPending } = useServerAction(createUserAction);

  // Client-side validation with same schema
  const form = useForm<CreateUserInput>({
    resolver: zodResolver(createUserSchema),
  });

  const onSubmit = async (data: CreateUserInput) => {
    // Data is already validated by React Hook Form
    const [result, err] = await execute(data);
    if (err) {
      // Handle server errors (like duplicate email)
      form.setError("root", { message: err.message });
    }
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* Form fields... */}
    </form>
  );
}
```

## Progressive Enhancement

```typescript
// components/progressive-form.tsx
"use client";

import { useServerAction } from "zsa-react";
import { submitAction } from "@/actions/form";

export function ProgressiveForm() {
  const { executeFormAction, isPending, isSuccess } =
    useServerAction(submitAction);

  return (
    // Works without JavaScript via native form submission
    // Enhanced with JavaScript for better UX
    <form action={executeFormAction} method="POST">
      <input name="email" type="email" required />
      <input name="name" required />
      
      <button type="submit" disabled={isPending}>
        {isPending ? "Submitting..." : "Submit"}
      </button>
      
      {/* Only shows with JavaScript */}
      {isSuccess && <p>Thank you!</p>}
    </form>
  );
}
```