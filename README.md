# German B1 Listening Comprehension App

An interactive web application designed to help German B1 learners practice their listening comprehension skills. The app generates stories with accompanying audio and provides comprehension questions to test understanding.

## Features

- **Interactive Chat Interface**: User-friendly chat-based interface for a natural learning experience
- **Audio Stories**: Listen to German stories at B1 level
- **Multiple Choice & Text Answers**: Choose between multiple choice or text-based answers
- **Immediate Feedback**: Get instant evaluation of your answers
- **Topic Selection**: Practice with stories from various topics
- **Responsive Design**: Works on both desktop and mobile devices

## Prerequisites

- Node.js (v16 or higher)
- npm (v8 or higher) or yarn
- Git (for version control)

## Getting Started

### Installation

1. Clone the repository:

   ```bash
   git clone https://github.com/wolf-utz/listen-up-chat
   cd listen-up-chat
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

### Running the Application

#### Development Mode

To start the development server:

```bash
npm run dev
```

Open your browser and navigate to `http://localhost:5173`

#### Production Build

To create a production build:

```bash
npm run build
```

The built files will be available in the `dist` directory.

## Project Structure

```
src/
├── assets/          # Static assets
├── components/      # Vue components
│   └── AudioPlayer.vue  # Audio player component
├── App.vue          # Main application component
└── main.ts          # Application entry point
```

## Technologies Used

- Vue 3 with Composition API
- TypeScript
- Vite (Build Tool)
- HTML5 Audio API
- CSS3 (with Flexbox and Grid)

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Vue.js and Vite teams for the excellent development experience
- The German language learning community for inspiration
