# Image Carousel / Slider

## Problem
Create an image carousel with auto-play, manual prev/next navigation, dots indicator, and swipe support.

## Key Concepts
- `useState` for current slide index
- `useEffect` for auto-play interval
- Touch events for swipe gestures
- CSS transforms for smooth transitions

## MANG-Level Expectations
- **UX**: Smooth transitions, infinite loop, keyboard navigation
- **Performance**: Lazy loading images, preload adjacent slides
- **Mobile**: Touch/swipe gestures, responsive design
- **Accessibility**: ARIA labels, keyboard support, pause on focus
- **Features**: Thumbnails, captions, fullscreen mode

## Solution

```jsx
import { useState, useEffect, useRef, useCallback } from 'react';

/**
 * TRICKY PART #1: Infinite loop carousel
 * - Use modulo operator to wrap around
 * - MANG expectation: Handle edge cases (0 images, 1 image)
 */
function Carousel({ 
  images, 
  autoPlay = true, 
  interval = 3000,
  showDots = true,
  showArrows = true 
}) {
  const [currentIndex, setCurrentIndex] = useState(0);
  const [isPlaying, setIsPlaying] = useState(autoPlay);
  const [touchStart, setTouchStart] = useState(0);
  const [touchEnd, setTouchEnd] = useState(0);
  const timerRef = useRef(null);

  const totalSlides = images.length;

  /**
   * TRICKY PART #2: Auto-play with cleanup
   * - Use useRef to store interval ID
   * - MANG will ask: "How do you prevent memory leaks?"
   * - CRITICAL: Clear interval on unmount and when dependencies change
   */
  useEffect(() => {
    if (!isPlaying || totalSlides <= 1) return;

    timerRef.current = setInterval(() => {
      goToNext();
    }, interval);

    return () => {
      if (timerRef.current) {
        clearInterval(timerRef.current);
      }
    };
  }, [isPlaying, currentIndex, interval, totalSlides]);

  /**
   * TRICKY PART #3: Navigation with wrapping
   * - Handle boundary conditions (first/last slide)
   * - MANG expectation: Smooth infinite loop
   */
  const goToNext = useCallback(() => {
    setCurrentIndex(prev => (prev + 1) % totalSlides);
  }, [totalSlides]);

  const goToPrev = useCallback(() => {
    setCurrentIndex(prev => (prev - 1 + totalSlides) % totalSlides);
  }, [totalSlides]);

  const goToSlide = useCallback((index) => {
    setCurrentIndex(index);
  }, []);

  /**
   * TRICKY PART #4: Touch/Swipe gestures
   * - Calculate swipe distance and direction
   * - MANG will ask: "How do you handle accidental swipes?"
   * - Use minimum swipe distance threshold
   */
  const handleTouchStart = (e) => {
    setTouchStart(e.targetTouches[0].clientX);
  };

  const handleTouchMove = (e) => {
    setTouchEnd(e.targetTouches[0].clientX);
  };

  const handleTouchEnd = () => {
    if (!touchStart || !touchEnd) return;
    
    const distance = touchStart - touchEnd;
    const minSwipeDistance = 50; // Minimum distance for swipe

    if (Math.abs(distance) < minSwipeDistance) return;

    if (distance > 0) {
      // Swiped left - go to next
      goToNext();
    } else {
      // Swiped right - go to previous
      goToPrev();
    }

    // Reset
    setTouchStart(0);
    setTouchEnd(0);
  };

  /**
   * TRICKY PART #5: Keyboard navigation
   * - Arrow keys for navigation
   * - Space to pause/play
   * - MANG expectation: Accessibility compliance
   */
  useEffect(() => {
    const handleKeyDown = (e) => {
      switch (e.key) {
        case 'ArrowLeft':
          goToPrev();
          break;
        case 'ArrowRight':
          goToNext();
          break;
        case ' ':
          e.preventDefault();
          setIsPlaying(prev => !prev);
          break;
        case 'Home':
          goToSlide(0);
          break;
        case 'End':
          goToSlide(totalSlides - 1);
          break;
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [goToNext, goToPrev, goToSlide, totalSlides]);

  // Pause on hover
  const handleMouseEnter = () => {
    if (autoPlay) setIsPlaying(false);
  };

  const handleMouseLeave = () => {
    if (autoPlay) setIsPlaying(true);
  };

  if (!images || images.length === 0) {
    return <div className="carousel-empty">No images to display</div>;
  }

  return (
    <div
      className="carousel"
      onMouseEnter={handleMouseEnter}
      onMouseLeave={handleMouseLeave}
      onTouchStart={handleTouchStart}
      onTouchMove={handleTouchMove}
      onTouchEnd={handleTouchEnd}
      role="region"
      aria-label="Image carousel"
      aria-live="polite"
    >
      {/* Slides */}
      <div className="carousel-slides">
        {images.map((image, index) => (
          <div
            key={index}
            className={`carousel-slide ${index === currentIndex ? 'active' : ''}`}
            style={{
              transform: `translateX(${(index - currentIndex) * 100}%)`
            }}
          >
            <img
              src={image.src}
              alt={image.alt || `Slide ${index + 1}`}
              loading={Math.abs(index - currentIndex) <= 1 ? 'eager' : 'lazy'}
            />
            {image.caption && (
              <div className="carousel-caption">{image.caption}</div>
            )}
          </div>
        ))}
      </div>

      {/* Previous/Next Arrows */}
      {showArrows && totalSlides > 1 && (
        <>
          <button
            className="carousel-arrow carousel-arrow-prev"
            onClick={goToPrev}
            aria-label="Previous slide"
          >
            ‹
          </button>
          <button
            className="carousel-arrow carousel-arrow-next"
            onClick={goToNext}
            aria-label="Next slide"
          >
            ›
          </button>
        </>
      )}

      {/* Dots Indicator */}
      {showDots && totalSlides > 1 && (
        <div className="carousel-dots">
          {images.map((_, index) => (
            <button
              key={index}
              className={`carousel-dot ${index === currentIndex ? 'active' : ''}`}
              onClick={() => goToSlide(index)}
              aria-label={`Go to slide ${index + 1}`}
              aria-current={index === currentIndex}
            />
          ))}
        </div>
      )}

      {/* Play/Pause Button */}
      {autoPlay && totalSlides > 1 && (
        <button
          className="carousel-play-pause"
          onClick={() => setIsPlaying(prev => !prev)}
          aria-label={isPlaying ? 'Pause' : 'Play'}
        >
          {isPlaying ? '⏸' : '▶'}
        </button>
      )}

      {/* Slide Counter */}
      <div className="carousel-counter" aria-live="polite">
        {currentIndex + 1} / {totalSlides}
      </div>
    </div>
  );
}

export default Carousel;
```

## CSS

```css
.carousel {
  position: relative;
  width: 100%;
  max-width: 800px;
  margin: 0 auto;
  overflow: hidden;
  border-radius: 8px;
}

.carousel-slides {
  position: relative;
  width: 100%;
  padding-bottom: 56.25%; /* 16:9 aspect ratio */
}

.carousel-slide {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transition: transform 0.5s ease-in-out;
  opacity: 0;
}

.carousel-slide.active {
  opacity: 1;
}

.carousel-slide img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.carousel-caption {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 1rem;
  text-align: center;
}

.carousel-arrow {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  background: rgba(0, 0, 0, 0.5);
  color: white;
  border: none;
  font-size: 2rem;
  padding: 1rem;
  cursor: pointer;
  z-index: 10;
  transition: background 0.3s;
}

.carousel-arrow:hover {
  background: rgba(0, 0, 0, 0.8);
}

.carousel-arrow-prev {
  left: 1rem;
}

.carousel-arrow-next {
  right: 1rem;
}

.carousel-dots {
  position: absolute;
  bottom: 1rem;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 0.5rem;
  z-index: 10;
}

.carousel-dot {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  border: 2px solid white;
  background: transparent;
  cursor: pointer;
  transition: background 0.3s;
}

.carousel-dot.active {
  background: white;
}

.carousel-play-pause {
  position: absolute;
  top: 1rem;
  right: 1rem;
  background: rgba(0, 0, 0, 0.5);
  color: white;
  border: none;
  padding: 0.5rem;
  cursor: pointer;
  border-radius: 4px;
  z-index: 10;
}

.carousel-counter {
  position: absolute;
  top: 1rem;
  left: 1rem;
  background: rgba(0, 0, 0, 0.5);
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  font-size: 0.875rem;
  z-index: 10;
}

.carousel-empty {
  padding: 2rem;
  text-align: center;
  color: #666;
}
```

## Usage

```jsx
const images = [
  { src: '/images/slide1.jpg', alt: 'Slide 1', caption: 'Beautiful sunset' },
  { src: '/images/slide2.jpg', alt: 'Slide 2', caption: 'Mountain view' },
  { src: '/images/slide3.jpg', alt: 'Slide 3', caption: 'Ocean waves' }
];

<Carousel
  images={images}
  autoPlay={true}
  interval={3000}
  showDots={true}
  showArrows={true}
/>
```

## Test Cases

```jsx
import { render, screen, fireEvent, act } from '@testing-library/react';
import Carousel from './Carousel';

jest.useFakeTimers();

const mockImages = [
  { src: 'image1.jpg', alt: 'Image 1' },
  { src: 'image2.jpg', alt: 'Image 2' },
  { src: 'image3.jpg', alt: 'Image 3' }
];

describe('Carousel', () => {
  afterEach(() => {
    jest.clearAllTimers();
  });

  test('renders all images', () => {
    render(<Carousel images={mockImages} autoPlay={false} />);
    
    expect(screen.getByAltText('Image 1')).toBeInTheDocument();
    expect(screen.getByAltText('Image 2')).toBeInTheDocument();
    expect(screen.getByAltText('Image 3')).toBeInTheDocument();
  });

  test('shows first image initially', () => {
    render(<Carousel images={mockImages} autoPlay={false} />);
    
    const firstSlide = screen.getByAltText('Image 1').closest('.carousel-slide');
    expect(firstSlide).toHaveClass('active');
  });

  test('navigates to next slide on next button click', () => {
    render(<Carousel images={mockImages} autoPlay={false} />);
    
    fireEvent.click(screen.getByLabelText('Next slide'));
    
    const secondSlide = screen.getByAltText('Image 2').closest('.carousel-slide');
    expect(secondSlide).toHaveClass('active');
  });

  test('navigates to previous slide on prev button click', () => {
    render(<Carousel images={mockImages} autoPlay={false} />);
    
    fireEvent.click(screen.getByLabelText('Previous slide'));
    
    const lastSlide = screen.getByAltText('Image 3').closest('.carousel-slide');
    expect(lastSlide).toHaveClass('active');
  });

  test('wraps around to first slide from last', () => {
    render(<Carousel images={mockImages} autoPlay={false} />);
    
    fireEvent.click(screen.getByLabelText('Next slide'));
    fireEvent.click(screen.getByLabelText('Next slide'));
    fireEvent.click(screen.getByLabelText('Next slide'));
    
    const firstSlide = screen.getByAltText('Image 1').closest('.carousel-slide');
    expect(firstSlide).toHaveClass('active');
  });

  test('auto-plays slides', () => {
    render(<Carousel images={mockImages} autoPlay={true} interval={1000} />);
    
    act(() => {
      jest.advanceTimersByTime(1000);
    });
    
    const secondSlide = screen.getByAltText('Image 2').closest('.carousel-slide');
    expect(secondSlide).toHaveClass('active');
  });

  test('pauses on hover', () => {
    render(<Carousel images={mockImages} autoPlay={true} interval={1000} />);
    
    const carousel = screen.getByRole('region');
    fireEvent.mouseEnter(carousel);
    
    act(() => {
      jest.advanceTimersByTime(2000);
    });
    
    const firstSlide = screen.getByAltText('Image 1').closest('.carousel-slide');
    expect(firstSlide).toHaveClass('active');
  });

  test('resumes on mouse leave', () => {
    render(<Carousel images={mockImages} autoPlay={true} interval={1000} />);
    
    const carousel = screen.getByRole('region');
    fireEvent.mouseEnter(carousel);
    fireEvent.mouseLeave(carousel);
    
    act(() => {
      jest.advanceTimersByTime(1000);
    });
    
    const secondSlide = screen.getByAltText('Image 2').closest('.carousel-slide');
    expect(secondSlide).toHaveClass('active');
  });

  test('navigates with dot indicators', () => {
    render(<Carousel images={mockImages} autoPlay={false} showDots={true} />);
    
    const dots = screen.getAllByRole('button', { name: /Go to slide/ });
    fireEvent.click(dots[2]);
    
    const thirdSlide = screen.getByAltText('Image 3').closest('.carousel-slide');
    expect(thirdSlide).toHaveClass('active');
  });

  test('navigates with keyboard arrows', () => {
    render(<Carousel images={mockImages} autoPlay={false} />);
    
    fireEvent.keyDown(window, { key: 'ArrowRight' });
    
    const secondSlide = screen.getByAltText('Image 2').closest('.carousel-slide');
    expect(secondSlide).toHaveClass('active');
    
    fireEvent.keyDown(window, { key: 'ArrowLeft' });
    
    const firstSlide = screen.getByAltText('Image 1').closest('.carousel-slide');
    expect(firstSlide).toHaveClass('active');
  });

  test('toggles play/pause', () => {
    render(<Carousel images={mockImages} autoPlay={true} interval={1000} />);
    
    const playPauseBtn = screen.getByLabelText('Pause');
    fireEvent.click(playPauseBtn);
    
    expect(screen.getByLabelText('Play')).toBeInTheDocument();
    
    act(() => {
      jest.advanceTimersByTime(2000);
    });
    
    const firstSlide = screen.getByAltText('Image 1').closest('.carousel-slide');
    expect(firstSlide).toHaveClass('active');
  });

  test('shows slide counter', () => {
    render(<Carousel images={mockImages} autoPlay={false} />);
    
    expect(screen.getByText('1 / 3')).toBeInTheDocument();
    
    fireEvent.click(screen.getByLabelText('Next slide'));
    
    expect(screen.getByText('2 / 3')).toBeInTheDocument();
  });

  test('handles empty images array', () => {
    render(<Carousel images={[]} />);
    
    expect(screen.getByText('No images to display')).toBeInTheDocument();
  });

  test('handles single image', () => {
    render(<Carousel images={[mockImages[0]]} />);
    
    expect(screen.queryByLabelText('Next slide')).not.toBeInTheDocument();
    expect(screen.queryByLabelText('Previous slide')).not.toBeInTheDocument();
  });

  test('displays captions', () => {
    const imagesWithCaptions = [
      { src: 'image1.jpg', alt: 'Image 1', caption: 'Caption 1' }
    ];
    
    render(<Carousel images={imagesWithCaptions} autoPlay={false} />);
    
    expect(screen.getByText('Caption 1')).toBeInTheDocument();
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add thumbnail navigation?**
```jsx
<div className="carousel-thumbnails">
  {images.map((image, index) => (
    <button
      key={index}
      className={`thumbnail ${index === currentIndex ? 'active' : ''}`}
      onClick={() => goToSlide(index)}
    >
      <img src={image.src} alt={`Thumbnail ${index + 1}`} />
    </button>
  ))}
</div>
```

**Q: How would you add lazy loading for images?**
```jsx
// Already implemented with loading attribute
<img
  src={image.src}
  alt={image.alt}
  loading={Math.abs(index - currentIndex) <= 1 ? 'eager' : 'lazy'}
/>

// Or use Intersection Observer for more control
const [loadedImages, setLoadedImages] = useState(new Set([0]));

useEffect(() => {
  // Preload adjacent images
  const toLoad = [currentIndex - 1, currentIndex, currentIndex + 1]
    .map(i => (i + totalSlides) % totalSlides);
  
  setLoadedImages(prev => new Set([...prev, ...toLoad]));
}, [currentIndex, totalSlides]);
```

**Q: How would you add fullscreen mode?**
```jsx
const [isFullscreen, setIsFullscreen] = useState(false);

const toggleFullscreen = () => {
  if (!isFullscreen) {
    carouselRef.current?.requestFullscreen();
  } else {
    document.exitFullscreen();
  }
  setIsFullscreen(!isFullscreen);
};

<button onClick={toggleFullscreen}>
  {isFullscreen ? 'Exit Fullscreen' : 'Fullscreen'}
</button>
```

**Q: How would you add zoom on image click?**
```jsx
const [zoomedImage, setZoomedImage] = useState(null);

<img
  src={image.src}
  onClick={() => setZoomedImage(image)}
  style={{ cursor: 'zoom-in' }}
/>

{zoomedImage && (
  <div className="image-zoom-overlay" onClick={() => setZoomedImage(null)}>
    <img src={zoomedImage.src} alt={zoomedImage.alt} />
  </div>
)}
```

**Q: How would you optimize for performance with many images?**
```jsx
// 1. Virtual carousel - only render visible + adjacent slides
// 2. Use CSS transform instead of re-rendering
// 3. Memoize slide components
const Slide = React.memo(({ image, isActive }) => (
  <div className={`slide ${isActive ? 'active' : ''}`}>
    <img src={image.src} alt={image.alt} />
  </div>
));

// 4. Use requestAnimationFrame for smooth animations
// 5. Debounce swipe gestures
```

## Edge Cases Handled
- Empty images array
- Single image (hide navigation)
- Infinite loop (wraps around)
- Auto-play with pause on hover
- Touch/swipe gestures with minimum distance
- Keyboard navigation (arrows, space, home, end)
- Lazy loading for performance
- Play/pause toggle
- Slide counter
- Captions support
- ARIA attributes for accessibility
- Cleanup timers on unmount
