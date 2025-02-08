---
layout: post
title: "Image Crop Tool using SwiftUI"
date: 2025-02-08
categories: ios, swift, swiftui
author: "Alan"
---

# Image Cropping Application Tutorial

This tutorial explains how to implement an image cropping feature in SwiftUI using a Model-View-ViewModel (MVVM) architecture. The application allows users to select images, crop them with a draggable overlay, and save the results.

Future enhancements could include aspect ratio controls, rotation, or more advanced image processing features.

## Architecture Overview

The application consists of three main components:

1. `ImagePickerView` - Main view handling UI and user interactions
2. `ImagePickerViewModel` - Business logic and state management
3. `CropOverlayView` - Custom view for the cropping interface

## 1. Image Selection and Display

The `ImagePickerView` uses SwiftUI's `PhotosPicker` for image selection:

```swift
PhotosPicker(selection: $selectedItem, matching: .images, photoLibrary: .shared()) {
    Label("Select", systemImage: "photo")
}
.buttonStyle(.borderedProminent)
```

When an image is selected, it's processed by the ViewModel:

```swift
func loadImage(from item: PhotosPickerItem?) async {
    guard let item = item else { return }

    croppedImage = nil
    cachedDisplaySize = nil  // Reset cache when loading new image

    do {
        if let data = try await item.loadTransferable(type: Data.self),
           let uiImage = UIImage(data: data) {
            selectedImage = uiImage
        }
    } catch {
        print("Failed to load image: \(error.localizedDescription)")
    }
}
```

## 2. Crop Overlay Implementation

The `CropOverlayView` creates a draggable crop rectangle with a semi-transparent overlay:

```swift
struct CropOverlayView: View {
    @Binding var cropRect: CGRect

    var body: some View {
        GeometryReader { geometry in
            ZStack {
                // Semi-transparent overlay
                Color.black.opacity(0.5)
                    .mask(
                        Rectangle()
                            .overlay(
                                Rectangle()
                                    .frame(width: cropRect.width, height: cropRect.height)
                                    .position(x: cropRect.midX, y: cropRect.midY)
                                    .blendMode(.destinationOut)
                            )
                    )

                // Crop rectangle border
                Rectangle()
                    .strokeBorder(Color.white, lineWidth: 2)
                    .frame(width: cropRect.width, height: cropRect.height)
                    .position(x: cropRect.midX, y: cropRect.midY)
            }
        }
    }
}
```

## 3. Crop Mode Management

The application uses a `isCropModeActive` state to manage the cropping interface:

```swift
func toggleCropMode() {
    isCropModeActive.toggle()
    if !isCropModeActive {
        // Reset crop rect when exiting crop mode
        cropRect = .zero
    }
}
```

## 4. Image Cropping Logic

The crucial cropping implementation in the ViewModel:

```swift
func cropImage() {
    guard let image = selectedImage else { return }

    let displaySize = getDisplaySize(for: image)

    // Calculate scaling factor from display size to actual image size
    let scaleX = image.size.width / displaySize.width
    let scaleY = image.size.height / displaySize.height

    // Convert crop rect to image coordinates
    let imageCropRect = CGRect(
        x: cropRect.origin.x * scaleX,
        y: cropRect.origin.y * scaleY,
        width: cropRect.width * scaleX,
        height: cropRect.height * scaleY
    )

    // Ensure crop rect is within image bounds
    let imageBounds = CGRect(origin: .zero, size: image.size)
    let validCropRect = imageCropRect.intersection(imageBounds)

    if let cgImage = image.cgImage,
       let croppedCGImage = cgImage.cropping(to: validCropRect) {
        croppedImage = UIImage(cgImage: croppedCGImage)
    }
}
```

## 5. Display Size Calculations

The application maintains proper scaling between display and actual image sizes:

```swift
private func getDisplaySize(
    for image: UIImage,
    containerWidth: CGFloat = UIScren.main.bounds.width
) -> CGSize {
    if let cached = cachedDisplaySize {
        return cached
    }
    let imageAspect = image.size.width / image.size.height
    let displayHeight = min(400, containerWidth / imageAspect)
    let displayWidth = displayHeight * imageAspect
    let size = CGSize(width: displayWidth, height: displayHeight)
    cachedDisplaySize = size
    return size
}
```

## Key Features

1. **Image Selection**: Uses PhotosPicker for native iOS photo library access
2. **Interactive Cropping**: Draggable crop rectangle with visual feedback
3. **Proper Scaling**: Maintains correct proportions between display and actual image sizes
4. **State Management**: Clear separation of concerns with MVVM architecture
5. **Crop Mode**: Dedicated mode for cropping operations with appropriate UI states

## Implementation Considerations

1. **Performance**: Display size calculations are cached for efficiency
2. **Error Handling**: Basic error handling for image loading failures
3. **UI/UX**: Clear mode transitions and intuitive controls
4. **Memory Management**: Proper cleanup of resources when loading new images
5. **Constraints**: Crop rectangle is constrained within image bounds

This implementation provides a robust foundation for image cropping functionality that can be extended with additional features like aspect ratio controls, rotation, or more sophisticated image processing capabilities.
